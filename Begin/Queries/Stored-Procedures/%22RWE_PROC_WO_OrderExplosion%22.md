CREATE PROCEDURE "RWE_PROC_WO_OrderExplosion" (
    IN  IV_PRODORDER_DOCENTRY INTEGER,  -- DocEntry OWOR; NULL = все верхнеуровневые заказы
    OUT OT_RESULT TABLE (
        "SalesOrderDocEntry"      INTEGER,
        "SalesOrderDocNum"        INTEGER,
        "BomDepth"                INTEGER,       -- 0 = сам заказ на готовую продукцию
        "ItemCode"                NVARCHAR(50),
        "ItemName"                NVARCHAR(200),
        "WhsCode"                 NVARCHAR(8),
        "RequiredQty"             DECIMAL(19,6), -- WOR1."PlannedQty" (уже итоговое, не на единицу)
        "UomCode"                 NVARCHAR(10),
        "ParentItemCode"          NVARCHAR(50),
        "ParentProdOrderDocEntry" INTEGER,
        "ParentProdOrderDocNum"   INTEGER,
        "ProdOrderDocEntry"       INTEGER,       -- заполнено, если для ЭТОЙ позиции есть реальный производственный заказ
        "ProdOrderDocNum"         INTEGER,
        "ProdOrderStatusRaw"      NVARCHAR(1),   -- 'P'/'R'/'L' - для диагностики
        "PostDate"                DATE,
        "DueDate"                 DATE,
        "Status"                  NVARCHAR(60)   -- одно из 5 текстовых значений, см. примечание выше
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN
    DECLARE lv_iter           INTEGER := 0;
    DECLARE lv_maxiter        INTEGER := 50;     -- защита от зацикливания при некорректных/циклических связях
    DECLARE lv_treetype       NVARCHAR(1) := 'P'; -- код "производственного" BoM в OITT.TreeType (см. MRP_Quotation_Explosion)
    DECLARE lv_origintype_so  NVARCHAR(1) := 'S'; -- OWOR.OriginType для связи с Заказом на продажу (подтверждено) - используется только для справочных колонок SalesOrderDocEntry/DocNum
    DECLARE lv_prodorder_obj  INTEGER     := 202;  -- код объекта "Производственный заказ" в SAP B1 (подтверждено)
    DECLARE lv_wave_cnt       INTEGER := 0;
    DECLARE lv_continue       INTEGER := 1;

    -- ------------------------------------------------------------------
    -- Глубина 0: либо один конкретный производственный заказ
    -- (IV_PRODORDER_DOCENTRY передан), либо ВСЕ "верхнеуровневые" заказы в системе (IV_PRODORDER_DOCENTRY = NULL) - то есть те, что не
    -- встречаются ни у кого в WOR1."PoDocEntry" как чей-то дочерний заказ. SalesOrderDocEntry/DocNum заполняются НЕ из входного параметра, а
    -- по факту наличия связи у САМОГО найденного заказа (OriginType='S') если связи с Заказом на продажу нет (например, создан вручную),
    -- будут NULL.
      -- ------------------------------------------------------------------
    lt_wave =
        SELECT
            CASE WHEN O."OriginType" = :lv_origintype_so THEN O."OriginAbs" ELSE CAST(NULL AS INTEGER) END AS "SalesOrderDocEntry",
            CASE WHEN O."OriginType" = :lv_origintype_so THEN O."OriginNum" ELSE CAST(NULL AS INTEGER) END AS "SalesOrderDocNum",
            0                       AS "BomDepth",
            O."ItemCode",
            IM."ItemName",
            O."Warehouse"           AS "WhsCode",
            O."PlannedQty"          AS "RequiredQty",
            O."Uom"                 AS "UomCode",
            CAST(NULL AS NVARCHAR(50)) AS "ParentItemCode",
            CAST(NULL AS INTEGER)      AS "ParentProdOrderDocEntry",
            CAST(NULL AS INTEGER)      AS "ParentProdOrderDocNum",
            O."DocEntry"            AS "ProdOrderDocEntry",
            O."DocNum"              AS "ProdOrderDocNum",
            O."Status"              AS "ProdOrderStatusRaw",
            O."PostDate",
            O."DueDate",
            CASE O."Status"
                WHEN 'P' THEN 'Произв.заказ создан, Planned'
                WHEN 'R' THEN 'Произв.заказ создан, Released'
                WHEN 'L' THEN 'Произв.заказ создан, Closed'
                ELSE 'Произв.заказ создан, ' || COALESCE(O."Status", '?')
            END AS "Status"
          FROM "OWOR" O
          JOIN "OITM" IM ON IM."ItemCode" = O."ItemCode"
         WHERE (:iv_prodorder_docentry IS NOT NULL AND O."DocEntry" = :iv_prodorder_docentry)
            OR (:iv_prodorder_docentry IS NULL AND NOT EXISTS (
                    SELECT 1 FROM "WOR1" W2
                     WHERE W2."PoDocType" = :lv_prodorder_obj
                       AND W2."PoDocEntry" = O."DocEntry"
                ));

    -- накопитель результата - структура один в один как у lt_wave/lt_next
    lt_out =
        SELECT
            CAST(NULL AS INTEGER)       AS "SalesOrderDocEntry",
            CAST(NULL AS INTEGER)       AS "SalesOrderDocNum",
            CAST(NULL AS INTEGER)       AS "BomDepth",
            CAST(NULL AS NVARCHAR(50))  AS "ItemCode",
            CAST(NULL AS NVARCHAR(200)) AS "ItemName",
            CAST(NULL AS NVARCHAR(8))   AS "WhsCode",
            CAST(NULL AS DECIMAL(19,6)) AS "RequiredQty",
            CAST(NULL AS NVARCHAR(10))  AS "UomCode",
            CAST(NULL AS NVARCHAR(50))  AS "ParentItemCode",
            CAST(NULL AS INTEGER)       AS "ParentProdOrderDocEntry",
            CAST(NULL AS INTEGER)       AS "ParentProdOrderDocNum",
            CAST(NULL AS INTEGER)       AS "ProdOrderDocEntry",
            CAST(NULL AS INTEGER)       AS "ProdOrderDocNum",
            CAST(NULL AS NVARCHAR(1))   AS "ProdOrderStatusRaw",
            CAST(NULL AS DATE)          AS "PostDate",
            CAST(NULL AS DATE)          AS "DueDate",
            CAST(NULL AS NVARCHAR(60))  AS "Status"
          FROM "DUMMY"
         WHERE 1 = 0;

    lt_out =
        SELECT * FROM :lt_out
        UNION ALL
        SELECT * FROM :lt_wave;

    -- ------------------------------------------------------------------
    -- Основной цикл: один проход = один уровень вложенности дерева РЕАЛЬНЫХ производственных заказов. Разворачиваем ТОЛЬКО те позиции
    -- предыдущей волны, у которых есть свой производственный заказ (ProdOrderDocEntry не пусто) - остальные (листья "не требуется" /
    -- "не создан") дальше не разворачиваем, разворачивать нечего.
    -- ------------------------------------------------------------------
    WHILE :lv_iter <= :lv_maxiter AND :lv_continue = 1 DO

        lt_next =
            SELECT
                PW."SalesOrderDocEntry",
                PW."SalesOrderDocNum",
                PW."BomDepth" + 1 AS "BomDepth",
                W."ItemCode",
                IM."ItemName",
                W."wareHouse"     AS "WhsCode",
                W."PlannedQty"    AS "RequiredQty",
                W."UomCode",
                PW."ItemCode"           AS "ParentItemCode",
                PW."ProdOrderDocEntry"  AS "ParentProdOrderDocEntry",
                PW."ProdOrderDocNum"    AS "ParentProdOrderDocNum",
                CASE WHEN W."PoDocType" = :lv_prodorder_obj THEN W."PoDocEntry" ELSE CAST(NULL AS INTEGER) END AS "ProdOrderDocEntry",
                CASE WHEN W."PoDocType" = :lv_prodorder_obj THEN W."PoDocNum"   ELSE CAST(NULL AS INTEGER) END AS "ProdOrderDocNum",
                CO."Status" AS "ProdOrderStatusRaw",
                CO."PostDate",
                CO."DueDate",
                CASE
                    WHEN BT."Code" IS NULL THEN 'Произв.заказ не требуется'
                    WHEN W."PoDocType" IS NULL OR W."PoDocType" <> :lv_prodorder_obj OR W."PoDocEntry" IS NULL
                        THEN 'Произв.заказ не создан'
                    WHEN CO."Status" = 'P' THEN 'Произв.заказ создан, Planned'
                    WHEN CO."Status" = 'R' THEN 'Произв.заказ создан, Released'
                    WHEN CO."Status" = 'L' THEN 'Произв.заказ создан, Closed'
                    ELSE 'Произв.заказ создан, ' || COALESCE(CO."Status", '?')
                END AS "Status"
              FROM :lt_wave PW
              JOIN "WOR1" W  ON W."DocEntry" = PW."ProdOrderDocEntry"
              JOIN "OITM" IM ON IM."ItemCode" = W."ItemCode"
              -- признак "у компонента есть своя спецификация" (полуфабрикат) MAX(Qauntity) защищает от задвоения, если у товара несколько строк OITT.
              LEFT JOIN (
                    SELECT "Code", MAX("Qauntity") AS "Qauntity"
                      FROM "OITT"
                     WHERE "TreeType" = :lv_treetype
                     GROUP BY "Code"
                   ) BT ON BT."Code" = W."ItemCode"
              LEFT JOIN "OWOR" CO
                ON CO."DocEntry" = W."PoDocEntry" AND W."PoDocType" = :lv_prodorder_obj;

        SELECT COUNT(*) INTO lv_wave_cnt FROM :lt_next;

        IF :lv_wave_cnt = 0 THEN
            lv_continue := 0;
        ELSE
            lt_out =
                SELECT * FROM :lt_out
                UNION ALL
                SELECT * FROM :lt_next;

            -- дальше разворачиваем только то, у чего реально есть свой заказ
            lt_wave =
                SELECT * FROM :lt_next
                 WHERE "ProdOrderDocEntry" IS NOT NULL;
        END IF;

        lv_iter := lv_iter + 1;
    END WHILE;

    OT_RESULT =
        SELECT *
          FROM :lt_out
         ORDER BY "BomDepth", "ParentProdOrderDocEntry", "ItemCode";

END;

-- ============================================================================
--
--   -- цепочка от конкретного производственного заказа:
--   CALL "PRODORDER_CHAIN_EXPLOSION"(<DocEntry производственного заказа>, ?);
--
--   -- отчёт по ВСЕМ верхнеуровневым производственным заказам в системе:
--   CALL "PRODORDER_CHAIN_EXPLOSION"(NULL, ?);
