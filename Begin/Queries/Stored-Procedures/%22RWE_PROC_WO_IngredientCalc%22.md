CREATE PROCEDURE "RWE_PROC_WO_IngredientCalc" (
    IN  IV_PRODORDER_DOCENTRY INTEGER,  -- DocEntry OWOR; NULL = все открытые заказы во всей системе
    OUT OT_RESULT TABLE (
        "Action"              NVARCHAR(20),   -- 'Переместить' или 'Закупить'
        "ItemCode"            NVARCHAR(50),
        "ItemName"            NVARCHAR(200),
        "ToWhsCode"           NVARCHAR(8),     -- склад назначения (нужен по спецификации/произв.заказу)
        "FromWhsCode"         NVARCHAR(8),     -- склад-источник; NULL для 'Закупить'
        "Qty"                 DECIMAL(19,6),   -- рекомендуемое количество (для 'Закупить' - уже округлено по OrdrMulti)
        "RawQty"              DECIMAL(19,6),   -- то же количество ДО округления по OrdrMulti (для 'Переместить' совпадает с Qty)
        "OrderMulti"          DECIMAL(19,6),   -- OITM."OrdrMulti", справочно (только 'Закупить')
        "RequiredQtyAtTarget" DECIMAL(19,6),   -- суммарная потребность на складе назначения (сумма по всем открытым заказам, за вычетом WOR1."IssuedQty")
        "AvailableAtTarget"   DECIMAL(19,6),   -- OnHand - Committed + OnOrder на складе назначения
        "ShortageAtTarget"    DECIMAL(19,6),   -- Required - Available на складе назначения, ДО учёта перемещений
        "AvailableAtSource"   DECIMAL(19,6)    -- только 'Переместить': OnHand - Committed на складе-источнике
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN
    DECLARE lv_iter          INTEGER := 0;
    DECLARE lv_maxiter       INTEGER := 50;
    DECLARE lv_continue      INTEGER := 1;
    DECLARE lv_wave_cnt      INTEGER := 0;
    DECLARE lv_treetype      NVARCHAR(1) := 'P';
    DECLARE lv_prodorder_obj INTEGER     := 202;

    -- ------------------------------------------------------------------
    -- 1) Область заказов: либо все открытые заказы системы, либо один
    --    заказ + вся его цепочка дочерних заказов вниз.
    -- ------------------------------------------------------------------
    IF :iv_prodorder_docentry IS NULL THEN

        lt_scope =
            SELECT "DocEntry" FROM "OWOR" WHERE "Status" <> 'L';

    ELSE

        lt_scope_wave =
            SELECT :iv_prodorder_docentry AS "DocEntry" FROM "DUMMY";

        lt_scope =
            SELECT "DocEntry" FROM :lt_scope_wave;

        WHILE :lv_iter <= :lv_maxiter AND :lv_continue = 1 DO

            lt_scope_next =
                SELECT DISTINCT W."PoDocEntry" AS "DocEntry"
                  FROM "WOR1" W
                  JOIN :lt_scope_wave SW ON SW."DocEntry" = W."DocEntry"
                 WHERE W."PoDocType" = :lv_prodorder_obj
                   AND W."PoDocEntry" IS NOT NULL;

            SELECT COUNT(*) INTO lv_wave_cnt FROM :lt_scope_next;

            IF :lv_wave_cnt = 0 THEN
                lv_continue := 0;
            ELSE
                lt_scope =
                    SELECT "DocEntry" FROM :lt_scope
                    UNION
                    SELECT "DocEntry" FROM :lt_scope_next;

                lt_scope_wave =
                    SELECT "DocEntry" FROM :lt_scope_next;
            END IF;

            lv_iter := lv_iter + 1;
        END WHILE;

        -- в области оставляем только НЕ закрытые заказы (закрытым уже
        -- ничего не нужно поставлять)
        lt_scope =
            SELECT S."DocEntry"
              FROM :lt_scope S
              JOIN "OWOR" O ON O."DocEntry" = S."DocEntry"
             WHERE O."Status" <> 'L';

    END IF;

    -- ------------------------------------------------------------------
    -- 2) Потребность в ЧИСТОМ СЫРЬЕ (компоненты без своей спецификации)
    --    по всем заказам области, за вычетом уже списанного (IssuedQty).
    -- ------------------------------------------------------------------
    lt_demand_raw =
        SELECT
            W."ItemCode",
            COALESCE(W."wareHouse", I."DfltWH") AS "WhsCode",
            CASE WHEN (W."PlannedQty" - W."IssuedQty") <= 0 THEN 0
                 ELSE (W."PlannedQty" - W."IssuedQty") END AS "ReqQty"
          FROM "WOR1" W
          JOIN :lt_scope S ON S."DocEntry" = W."DocEntry"
          JOIN "OITM" I ON I."ItemCode" = W."ItemCode"
          LEFT JOIN (
                SELECT "Code", MAX("Qauntity") AS "Qauntity"
                  FROM "OITT"
                 WHERE "TreeType" = :lv_treetype
                 GROUP BY "Code"
               ) BT ON BT."Code" = W."ItemCode"
         WHERE BT."Code" IS NULL;   -- только позиции без своей спецификации - чистые ингредиенты

    lt_demand =
        SELECT "ItemCode", "WhsCode", SUM("ReqQty") AS "RequiredQty"
          FROM :lt_demand_raw
         WHERE "WhsCode" IS NOT NULL
           AND "ReqQty" > 0
         GROUP BY "ItemCode", "WhsCode";

    -- ------------------------------------------------------------------
    -- 3) Остаток и нехватка на складе НАЗНАЧЕНИЯ (том, что требуется по
    --    спецификации/произв.заказу).
    -- ------------------------------------------------------------------
    lt_target =
        SELECT
            D."ItemCode",
            D."WhsCode" AS "ToWhsCode",
            D."RequiredQty",
            (COALESCE(WT."OnHand", 0) - COALESCE(WT."IsCommited", 0) + COALESCE(WT."OnOrder", 0)) AS "AvailableAtTarget",
            CASE
                WHEN D."RequiredQty" - (COALESCE(WT."OnHand", 0) - COALESCE(WT."IsCommited", 0) + COALESCE(WT."OnOrder", 0)) <= 0
                    THEN 0
                ELSE D."RequiredQty" - (COALESCE(WT."OnHand", 0) - COALESCE(WT."IsCommited", 0) + COALESCE(WT."OnOrder", 0))
            END AS "ShortageAtTarget"
          FROM :lt_demand D
          LEFT JOIN "OITW" WT
            ON WT."ItemCode" = D."ItemCode" AND WT."WhsCode" = D."WhsCode";

    -- ------------------------------------------------------------------
    -- 4) Кандидаты на перемещение - другие склады компании, где физически
    --    есть свободный (не зарезервированный) остаток этого товара.
    --    !!! Порядок ORDER BY ниже ("сначала самый большой остаток") -
    --    предположение, см. примечание "a)" в шапке файла.
    -- ------------------------------------------------------------------
    lt_sources =
        SELECT
            T."ItemCode", T."ToWhsCode", T."ShortageAtTarget",
            WS."WhsCode" AS "FromWhsCode",
            CASE WHEN (COALESCE(WS."OnHand", 0) - COALESCE(WS."IsCommited", 0)) <= 0 THEN 0
                 ELSE (COALESCE(WS."OnHand", 0) - COALESCE(WS."IsCommited", 0)) END AS "AvailableAtSource"
          FROM :lt_target T
          JOIN "OITW" WS
            ON WS."ItemCode" = T."ItemCode" AND WS."WhsCode" <> T."ToWhsCode"
         WHERE T."ShortageAtTarget" > 0;

    lt_sources =
        SELECT * FROM :lt_sources WHERE "AvailableAtSource" > 0;

    -- ------------------------------------------------------------------
    -- 5) Распределение нехватки по складам-источникам через бегущую сумму
    --    (оконная функция) - без WHILE-цикла, сразу по всем позициям.
    -- ------------------------------------------------------------------
    lt_alloc =
        SELECT
            "ItemCode", "ToWhsCode", "FromWhsCode", "AvailableAtSource", "ShortageAtTarget",
            SUM("AvailableAtSource") OVER (
                PARTITION BY "ItemCode", "ToWhsCode"
                ORDER BY "AvailableAtSource" DESC, "FromWhsCode"   -- !!! см. примечание "a)" в шапке файла
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) AS "RunningInclusive"
          FROM :lt_sources;

    -- ВАЖНО: HANA требует, чтобы у локальной табличной переменной структура
    -- (набор колонок) не менялась между переприсваиваниями в теле
    -- процедуры - иначе компиляция падает с "return type mismatch". Здесь
    -- набор колонок меняется (уходит "ShortageAtTarget"/"RunningInclusive",
    -- добавляется "TransferQty"), поэтому результат кладём в НОВУЮ
    -- переменную lt_alloc_final, а не переиспользуем lt_alloc.
    lt_alloc_final =
        SELECT
            "ItemCode", "ToWhsCode", "FromWhsCode", "AvailableAtSource",
            CASE
                WHEN ("ShortageAtTarget" - ("RunningInclusive" - "AvailableAtSource")) <= 0
                    THEN 0
                WHEN ("ShortageAtTarget" - ("RunningInclusive" - "AvailableAtSource")) >= "AvailableAtSource"
                    THEN "AvailableAtSource"
                ELSE "ShortageAtTarget" - ("RunningInclusive" - "AvailableAtSource")
            END AS "TransferQty"
          FROM :lt_alloc;

    lt_transfer_total =
        SELECT "ItemCode", "ToWhsCode", SUM("TransferQty") AS "TotalTransferred"
          FROM :lt_alloc_final
         GROUP BY "ItemCode", "ToWhsCode";

    -- ------------------------------------------------------------------
    -- 6) Остаток нехватки после всех перемещений -> рекомендация на
    --    закупку, округлённая вверх до кратного OITM."OrdrMulti".
    -- ------------------------------------------------------------------
    lt_purchase =
        SELECT
            T."ItemCode", T."ToWhsCode", T."RequiredQty", T."AvailableAtTarget", T."ShortageAtTarget",
            IM."ItemName", IM."OrdrMulti",
            CASE
                WHEN (T."ShortageAtTarget" - COALESCE(TT."TotalTransferred", 0)) <= 0 THEN 0
                ELSE T."ShortageAtTarget" - COALESCE(TT."TotalTransferred", 0)
            END AS "RawPurchaseQty"
          FROM :lt_target T
          JOIN "OITM" IM ON IM."ItemCode" = T."ItemCode"
          LEFT JOIN :lt_transfer_total TT
            ON TT."ItemCode" = T."ItemCode" AND TT."ToWhsCode" = T."ToWhsCode"
         WHERE T."ShortageAtTarget" > 0;

    -- та же причина, что и у lt_alloc_final выше - структура меняется
    -- (добавляется "PurchaseQty"), поэтому новая переменная, а не
    -- переприсваивание lt_purchase.
    lt_purchase_final =
        SELECT
            "ItemCode", "ToWhsCode", "RequiredQty", "AvailableAtTarget", "ShortageAtTarget",
            "ItemName", "OrdrMulti", "RawPurchaseQty",
            CASE
                WHEN "RawPurchaseQty" <= 0 THEN 0
                WHEN COALESCE("OrdrMulti", 0) > 0 THEN CEIL("RawPurchaseQty" / "OrdrMulti") * "OrdrMulti"
                ELSE "RawPurchaseQty"
            END AS "PurchaseQty"
          FROM :lt_purchase;

    -- ------------------------------------------------------------------
    -- 7) Итоговый результат: строки перемещения + строки закупки.
    -- ------------------------------------------------------------------
    lt_final =
        SELECT
            'Переместить' AS "Action",
            R."ItemCode", IM."ItemName",
            R."ToWhsCode", R."FromWhsCode",
            R."TransferQty" AS "Qty",
            R."TransferQty" AS "RawQty",
            CAST(NULL AS DECIMAL(19,6)) AS "OrderMulti",
            T."RequiredQty" AS "RequiredQtyAtTarget",
            T."AvailableAtTarget",
            T."ShortageAtTarget",
            R."AvailableAtSource"
          FROM :lt_alloc_final R
          JOIN :lt_target T ON T."ItemCode" = R."ItemCode" AND T."ToWhsCode" = R."ToWhsCode"
          JOIN "OITM" IM ON IM."ItemCode" = R."ItemCode"
         WHERE R."TransferQty" > 0

        UNION ALL

        SELECT
            'Закупить' AS "Action",
            P."ItemCode", P."ItemName",
            P."ToWhsCode", CAST(NULL AS NVARCHAR(8)) AS "FromWhsCode",
            P."PurchaseQty" AS "Qty",
            P."RawPurchaseQty" AS "RawQty",
            P."OrdrMulti" AS "OrderMulti",
            P."RequiredQty" AS "RequiredQtyAtTarget",
            P."AvailableAtTarget",
            P."ShortageAtTarget",
            CAST(NULL AS DECIMAL(19,6)) AS "AvailableAtSource"
          FROM :lt_purchase_final P
         WHERE P."PurchaseQty" > 0;

    OT_RESULT =
        SELECT *
          FROM :lt_final
         ORDER BY "ItemCode", "ToWhsCode", "Action" DESC, "FromWhsCode";

END;

