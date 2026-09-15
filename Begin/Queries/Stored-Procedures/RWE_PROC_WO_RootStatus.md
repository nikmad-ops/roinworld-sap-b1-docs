ALTER PROCEDURE "RWE_PROC_WO_RootStatus" (
    IN  IV_PRODORDER_DOCENTRY INTEGER,   -- DocEntry OWOR верхнего уровня; NULL = все незакрытые заказы верхнего уровня
    IN  IV_REPTYPE            NVARCHAR(1), -- 'D' = детальный отчёт; иначе (в т.ч. 'S'/NULL) = сводный отчёт
    OUT OT_RESULT TABLE (
        "RowType"                 NVARCHAR(10),  -- 'SUMMARY' или 'DETAIL' - какой отчёт сейчас в результате
        "RootProdOrderDocEntry"   INTEGER,        -- заказ верхнего уровня (в SUMMARY совпадает с ProdOrderDocEntry)
        "RootProdOrderDocNum"     INTEGER,
        "BomDepth"                INTEGER,        -- только DETAIL: 0 = сам заказ на готовую продукцию; в SUMMARY - NULL
        "ItemCode"                NVARCHAR(50),   -- SUMMARY: товар заказа верхнего уровня; DETAIL: товар этой строки
        "ItemName"                NVARCHAR(200),
        "WhsCode"                 NVARCHAR(8),    -- только DETAIL
        "RequiredQty"             DECIMAL(19,6),  -- только DETAIL
        "UomCode"                 NVARCHAR(10),   -- только DETAIL
        "ParentItemCode"          NVARCHAR(50),   -- только DETAIL
        "ParentProdOrderDocEntry" INTEGER,        -- только DETAIL
        "ParentProdOrderDocNum"   INTEGER,        -- только DETAIL
        "ProdOrderDocEntry"       INTEGER,        -- SUMMARY: сам заказ верхнего уровня; DETAIL: заказ НА ЭТУ позицию, если создан
        "ProdOrderDocNum"         INTEGER,
        "ProdOrderStatusRaw"      NVARCHAR(1),    -- 'P'/'R'/'L'
        "SalesOrderDocEntry"      INTEGER,        -- заполнено, только если у заказа верхнего уровня есть связь с Заказом на продажу
        "SalesOrderDocNum"        INTEGER,
        "PostDate"                DATE,
        "DueDate"                 DATE,
        "TotalComponentsChecked" INTEGER,        -- только SUMMARY: сколько компонентов всего проверено по всей цепочке вниз
        "MissingCount"            INTEGER,        -- только SUMMARY: сколько из них - недостающие произв.заказы
        "MissingItems"            NVARCHAR(2000), -- только SUMMARY: !!! см. примечание про STRING_AGG выше
        "MaxDepthReached"         INTEGER,        -- только SUMMARY: диагностика - до какого уровня дошло разузлование
        "Status"                  NVARCHAR(500)
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN
    DECLARE lv_iter          INTEGER := 0;
    DECLARE lv_maxiter       INTEGER := 50;      -- защита от зацикливания, как в PRODORDER_CHAIN_EXPLOSION
    DECLARE lv_treetype      NVARCHAR(1) := 'P';  -- код "производственного" BoM в OITT.TreeType
    DECLARE lv_origintype_so NVARCHAR(1) := 'S';  -- OWOR.OriginType для связи с Заказом на продажу
    DECLARE lv_prodorder_obj INTEGER     := 202;   -- код объекта "Производственный заказ"
    DECLARE lv_wave_cnt      INTEGER := 0;
    DECLARE lv_continue      INTEGER := 1;
    DECLARE lv_truncated     INTEGER := 0;         -- 1 = сработала защита lv_maxiter (см. примечание выше)

    -- ------------------------------------------------------------------
    -- Глубина 0: заказы ВЕРХНЕГО УРОВНЯ - без родительского произв.
    -- заказа. Если IV_PRODORDER_DOCENTRY передан - только этот заказ (и
    -- только если он сам верхнего уровня); если NULL - все незакрытые
    -- заказы верхнего уровня во всей системе.
    -- ------------------------------------------------------------------
    lt_wave =
        SELECT
            O."DocEntry"            AS "RootProdOrderDocEntry",
            O."DocNum"               AS "RootProdOrderDocNum",
            CASE WHEN O."OriginType" = :lv_origintype_so THEN O."OriginAbs" ELSE CAST(NULL AS INTEGER) END AS "SalesOrderDocEntry",
            CASE WHEN O."OriginType" = :lv_origintype_so THEN O."OriginNum" ELSE CAST(NULL AS INTEGER) END AS "SalesOrderDocNum",
            0                        AS "BomDepth",
            O."ItemCode",
            IM."ItemName",
            O."Warehouse"            AS "WhsCode",
            O."PlannedQty"           AS "RequiredQty",
            O."Uom"                  AS "UomCode",
            CAST(NULL AS NVARCHAR(50)) AS "ParentItemCode",
            CAST(NULL AS INTEGER)      AS "ParentProdOrderDocEntry",
            CAST(NULL AS INTEGER)      AS "ParentProdOrderDocNum",
            O."DocEntry"             AS "ProdOrderDocEntry",
            O."DocNum"               AS "ProdOrderDocNum",
            O."Status"               AS "ProdOrderStatusRaw",
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
         WHERE NOT EXISTS (
                    SELECT 1 FROM "WOR1" W2
                     WHERE W2."PoDocType" = :lv_prodorder_obj
                       AND W2."PoDocEntry" = O."DocEntry"
                )
           AND (
                  (:iv_prodorder_docentry IS NOT NULL AND O."DocEntry" = :iv_prodorder_docentry)
               OR (:iv_prodorder_docentry IS NULL AND O."Status" <> 'L')
               );

    -- накопитель детального результата - структура один в один как у
    -- lt_wave/lt_next
    lt_out =
        SELECT
            CAST(NULL AS INTEGER)       AS "RootProdOrderDocEntry",
            CAST(NULL AS INTEGER)       AS "RootProdOrderDocNum",
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
    -- Основной цикл: один проход = один уровень вложенности, СРАЗУ по
    -- ВСЕМ проверяемым заказам верхнего уровня параллельно (волновой
    -- обход, как в PRODORDER_CHAIN_EXPLOSION) - RootProdOrderDocEntry
    -- пронесён через все уровни, чтобы потом сгруппировать результат.
    -- ------------------------------------------------------------------
    WHILE :lv_iter <= :lv_maxiter AND :lv_continue = 1 DO

        lt_next =
            SELECT
                PW."RootProdOrderDocEntry",
                PW."RootProdOrderDocNum",
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

    IF :lv_continue = 1 THEN
        lv_truncated := 1;  -- цикл прерван по lv_maxiter, а не потому что дерево естественно закончилось
    END IF;

    -- ------------------------------------------------------------------
    -- Агрегация по каждому заказу верхнего уровня (нужна только для
    -- сводного отчёта, но считаем всегда - дёшево по сравнению с самим
    -- обходом дерева выше).
    -- ------------------------------------------------------------------
    lt_missing_agg =
        SELECT
            "RootProdOrderDocEntry",
            COUNT(*) AS "MissingCount",
            STRING_AGG("ItemCode", ', ') AS "MissingItems"  -- !!! см. примечание про STRING_AGG в шапке файла
          FROM :lt_out
         WHERE "Status" = 'Произв.заказ не создан'
         GROUP BY "RootProdOrderDocEntry";

    lt_scope_agg =
        SELECT
            "RootProdOrderDocEntry",
            COUNT(*)         AS "TotalComponentsChecked",  -- все проверенные компоненты на всех уровнях (BomDepth > 0)
            MAX("BomDepth")  AS "MaxDepthReached"
          FROM :lt_out
         WHERE "BomDepth" > 0
         GROUP BY "RootProdOrderDocEntry";

    -- ------------------------------------------------------------------
    -- Финальный выбор: ОДИН результирующий набор, вид которого зависит
    -- от IV_REPTYPE (см. примечание в шапке файла про ограничение В1).
    -- ------------------------------------------------------------------
    IF :iv_reptype = 'D' THEN

        OT_RESULT =
            SELECT
                'DETAIL'                        AS "RowType",
                "RootProdOrderDocEntry",
                "RootProdOrderDocNum",
                "BomDepth",
                "ItemCode",
                "ItemName",
                "WhsCode",
                "RequiredQty",
                "UomCode",
                "ParentItemCode",
                "ParentProdOrderDocEntry",
                "ParentProdOrderDocNum",
                "ProdOrderDocEntry",
                "ProdOrderDocNum",
                "ProdOrderStatusRaw",
                "SalesOrderDocEntry",
                "SalesOrderDocNum",
                "PostDate",
                "DueDate",
                CAST(NULL AS INTEGER)        AS "TotalComponentsChecked",
                CAST(NULL AS INTEGER)        AS "MissingCount",
                CAST(NULL AS NVARCHAR(2000)) AS "MissingItems",
                CAST(NULL AS INTEGER)        AS "MaxDepthReached",
                "Status"
              FROM :lt_out
             ORDER BY "RootProdOrderDocEntry", "BomDepth", "ItemCode";

    ELSE

        OT_RESULT =
            SELECT
                'SUMMARY'                     AS "RowType",
                R."RootProdOrderDocEntry",
                R."RootProdOrderDocNum",
                CAST(NULL AS INTEGER)         AS "BomDepth",
                R."ItemCode",
                R."ItemName",
                CAST(NULL AS NVARCHAR(8))     AS "WhsCode",
                CAST(NULL AS DECIMAL(19,6))   AS "RequiredQty",
                CAST(NULL AS NVARCHAR(10))    AS "UomCode",
                CAST(NULL AS NVARCHAR(50))    AS "ParentItemCode",
                CAST(NULL AS INTEGER)         AS "ParentProdOrderDocEntry",
                CAST(NULL AS INTEGER)         AS "ParentProdOrderDocNum",
                R."ProdOrderDocEntry",
                R."ProdOrderDocNum",
                R."ProdOrderStatusRaw",
                R."SalesOrderDocEntry",
                R."SalesOrderDocNum",
                R."PostDate",
                R."DueDate",
                COALESCE(SA."TotalComponentsChecked", 0) AS "TotalComponentsChecked",
                COALESCE(MA."MissingCount", 0)           AS "MissingCount",
                MA."MissingItems"                        AS "MissingItems",  -- !!! см. примечание про STRING_AGG в шапке файла
                SA."MaxDepthReached",
                CASE
                    WHEN :lv_truncated = 1
                        THEN 'ВНИМАНИЕ: достигнут предел глубины разворачивания (' || :lv_maxiter || ') - проверьте вручную, возможен цикл в данных'
                    WHEN COALESCE(MA."MissingCount", 0) = 0
                        THEN 'Все производственные заказы созданы до сырья'
                    ELSE 'Не хватает производственных заказов: ' || COALESCE(MA."MissingCount", 0) || ' поз. (' || COALESCE(MA."MissingItems", '?') || ')'
                END AS "Status"
              FROM :lt_out R
              LEFT JOIN :lt_missing_agg MA ON MA."RootProdOrderDocEntry" = R."RootProdOrderDocEntry"
              LEFT JOIN :lt_scope_agg   SA ON SA."RootProdOrderDocEntry" = R."RootProdOrderDocEntry"
             WHERE R."BomDepth" = 0
             ORDER BY R."RootProdOrderDocEntry";

    END IF;

END;

-- ============================================================================
-- ПРИМЕР ВЫЗОВА:
--
--   -- ежедневный сводный контроль по всем незакрытым заказам верхнего уровня:
--   CALL "PRODORDER_ROOT_STATUS"(NULL, 'S', ?);
--
--   -- детальный отчёт по всем незакрытым заказам верхнего уровня:
--   CALL "PRODORDER_ROOT_STATUS"(NULL, 'D', ?);
--
--   -- сводка/детали по конкретному заказу верхнего уровня (для теста):
--   CALL "PRODORDER_ROOT_STATUS"(8, 'S', ?);
--   CALL "PRODORDER_ROOT_STATUS"(8, 'D', ?);
--
-- Один знак "?" - под OT_RESULT (теперь только один выходной набор строк,
-- подходит для запуска как запрос/отчёт в В1).
--
-- Повторный деплой: используется CREATE OR REPLACE, поэтому просто заново
-- выполните этот файл. Если ваша версия HANA не поддерживает
-- CREATE OR REPLACE, выполните перед этим вручную:
--   DROP PROCEDURE "PRODORDER_ROOT_STATUS";
-- ============================================================================
