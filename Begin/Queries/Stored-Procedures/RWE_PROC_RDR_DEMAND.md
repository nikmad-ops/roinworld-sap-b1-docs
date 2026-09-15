CREATE PROCEDURE "RWE_PROC_RDR_DEMAND" (
    IN  IV_DOCENTRY INTEGER,
    OUT OT_RESULT   TABLE (
        "HierarchyLevel"     INTEGER,        -- 1 = закупить, 2 = произвести
        "ProductionSequence" INTEGER,        -- порядок создания произв. заказов (ASC), NULL для уровня 1
        "BomDepth"           INTEGER,        -- уровень вложенности BoM, на котором позиция была "закрыта" (0 = сама строка Quotation)
        "ItemCode"           NVARCHAR(50),
        "ItemName"           NVARCHAR(200),
        "WhsCode"            NVARCHAR(8),
        "RequiredQty"        DECIMAL(19,6),  -- суммарная потребность, агрегированная по всем родителям
        "OnHandQty"          DECIMAL(19,6),
        "CommittedQty"       DECIMAL(19,6),
        "OrderedQty"         DECIMAL(19,6),
        "AvailableQty"       DECIMAL(19,6),  -- OnHand - Committed + Ordered
        "ShortageQty"        DECIMAL(19,6),  -- рекомендуемое к закупке / производству количество
        "OrderMulti"         DECIMAL(19,6),  -- OITM."OrdrMulti" товара (для справки; реально применяется к округлению ShortageQty только если IsManufactured = 'N')
        "ParentItemCode"     NVARCHAR(1000), -- родитель(и), сгенерировавшие потребность (список, если их несколько)
        "SourceLineNum"      NVARCHAR(500),  -- строка(и)-источник в RDR1 (список)
        "IsManufactured"     NVARCHAR(1)     -- 'Y'/'N', диагностика
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN
    DECLARE lv_iter       INTEGER := 0;
    DECLARE lv_maxdepth   INTEGER := 0;
    DECLARE lv_maxiter    INTEGER := 50;   -- защита от зацикливания при некорректном/циклическом BoM
    DECLARE lv_treetype   NVARCHAR(1) := 'P'; -- код "производственного" BoM
    DECLARE lv_graph_iter INTEGER := 0;
    DECLARE lv_continue   INTEGER := 1;
    DECLARE lv_wave_cnt   INTEGER := 0;

    -- ------------------------------------------------------------------
    -- 2.1. Структурная развёртка - чтобы понять, какие товары встречаются в дереве и на какой максимальной
    --      глубине (low-level code, LLC). Это нужно, чтобы товар, который одновременно является и прямой строкой Quotation, и глубоким
    --      компонентом другого BoM, был "закрыт" (проверен по остаткам) только один раз - когда собраны ВСЕ его потребности.
    -- ------------------------------------------------------------------
    lt_graph_wave =
        SELECT DISTINCT
               Q."ItemCode",
               COALESCE(Q."WhsCode", I."DfltWH") AS "WhsCode",
               0 AS "Depth"
          FROM "RDR1" Q
          JOIN "OITM" I ON I."ItemCode" = Q."ItemCode"
         WHERE Q."DocEntry"   = :iv_docentry
           AND Q."LineStatus" = 'O'
           AND Q."ItemCode"  IS NOT NULL;

    lt_item_llc =
        SELECT
            CAST(NULL AS NVARCHAR(50)) AS "ItemCode",
            CAST(NULL AS NVARCHAR(8))  AS "WhsCode",
            CAST(NULL AS INTEGER)      AS "LLC"
          FROM "DUMMY"
         WHERE 1 = 0;

    WHILE :lv_graph_iter <= :lv_maxiter AND :lv_continue = 1 DO

        -- закладываем глубину текущей волны в LLC (берём максимум, если товар уже встречался мельче)
        lt_item_llc =
            SELECT "ItemCode", "WhsCode", MAX("Depth") AS "LLC"
              FROM (
                    SELECT "ItemCode", "WhsCode", "LLC" AS "Depth" FROM :lt_item_llc
                    UNION ALL
                    SELECT "ItemCode", "WhsCode", "Depth" FROM :lt_graph_wave
                   )
             GROUP BY "ItemCode", "WhsCode";

        -- следующая волна: компоненты BoM товаров текущей волны, у которых есть производственный BoM. Связь ITT1 -> OITT через
        -- ITT1."Father"; код самого компонента - ITT1."Code"
        lt_graph_wave =
            SELECT DISTINCT
                   C."Code" AS "ItemCode",
                   W."WhsCode",
                   W."Depth" + 1 AS "Depth"
              FROM :lt_graph_wave W
              JOIN "OITT" T ON T."Code" = W."ItemCode" AND T."TreeType" = :lv_treetype
              JOIN "ITT1" C ON C."Father" = T."Code";

        SELECT COUNT(*) INTO lv_wave_cnt FROM :lt_graph_wave;

        IF lv_wave_cnt = 0 THEN
            lv_continue := 0;
        END IF;

        lv_graph_iter := lv_graph_iter + 1;
    END WHILE;

    SELECT COALESCE(MAX("LLC"), 0) INTO lv_maxdepth FROM :lt_item_llc;

    -- ------------------------------------------------------------------
    -- 2.2. Аккумулятор потребности: по каждому (ItemCode, WhsCode) - бегущая сумма потребности, собранной со всех "родителей", которые уже были обработаны.
    -- ------------------------------------------------------------------
    lt_accum =
        SELECT
            CAST(NULL AS NVARCHAR(50))   AS "ItemCode",
            CAST(NULL AS NVARCHAR(8))    AS "WhsCode",
            CAST(NULL AS DECIMAL(19,6))  AS "ReqQty",
            CAST(NULL AS NVARCHAR(1000)) AS "ParentItemCode",
            CAST(NULL AS NVARCHAR(500))  AS "SourceLineNum"
          FROM "DUMMY"
         WHERE 1 = 0;

    -- "волна" глубины 0 - сами строки Sales Order. RDR1 не хранит единицу
    -- измерения строки напрямую, поэтому в качестве единицы строки берём
    -- единицу продажи товара по умолчанию (OITM."SUoMEntry") и переводим в
    -- базовую единицу группы UoM через MRP_FN_UOM_TO_BASEQTY (см. примечание
    -- в начале файла). На товарах, где SUoM = базовой единице сейчас у
    -- всех товаров пользователя так), это не меняет количество 
    lt_wave =
        SELECT
            Q."ItemCode",
            COALESCE(Q."WhsCode", I."DfltWH") AS "WhsCode",
            "RWE_UOM_TO_BASEQTY"(Q."ItemCode", I."SUoMEntry", Q."Quantity") AS "ReqQty",
            CAST(NULL AS NVARCHAR(50)) AS "ParentItemCode",
            Q."LineNum" AS "SourceLineNum"
          FROM "RDR1" Q
          JOIN "OITM" I ON I."ItemCode" = Q."ItemCode"
         WHERE Q."DocEntry"   = :iv_docentry
           AND Q."LineStatus" = 'O'
           AND Q."ItemCode"  IS NOT NULL;

    -- накопитель результата
    lt_out =
        SELECT
            CAST(NULL AS INTEGER)        AS "HierarchyLevel",
            CAST(NULL AS INTEGER)        AS "ProductionSequence",
            CAST(NULL AS INTEGER)        AS "BomDepth",
            CAST(NULL AS NVARCHAR(50))   AS "ItemCode",
            CAST(NULL AS NVARCHAR(200))  AS "ItemName",
            CAST(NULL AS NVARCHAR(8))    AS "WhsCode",
            CAST(NULL AS DECIMAL(19,6))  AS "RequiredQty",
            CAST(NULL AS DECIMAL(19,6))  AS "OnHandQty",
            CAST(NULL AS DECIMAL(19,6))  AS "CommittedQty",
            CAST(NULL AS DECIMAL(19,6))  AS "OrderedQty",
            CAST(NULL AS DECIMAL(19,6))  AS "AvailableQty",
            CAST(NULL AS DECIMAL(19,6))  AS "ShortageQty",
            CAST(NULL AS DECIMAL(19,6))  AS "OrderMulti",
            CAST(NULL AS NVARCHAR(1000)) AS "ParentItemCode",
            CAST(NULL AS NVARCHAR(500))  AS "SourceLineNum",
            CAST(NULL AS NVARCHAR(1))    AS "IsManufactured"
          FROM "DUMMY"
         WHERE 1 = 0;

    -- ------------------------------------------------------------------
    -- 2.3. Основной цикл: один проход = одна глубина BoM.
    -- ------------------------------------------------------------------
    WHILE :lv_iter <= :lv_maxdepth DO

        -- a) вливаем волну текущей глубины в аккумулятор.
        --    Суммы количества и список меток (родитель/строка-источник)
        --    считаются двумя отдельными агрегациями, дедупликация
        --    делается через SELECT DISTINCT в подзапросе.
        lt_union =
            SELECT "ItemCode", "WhsCode", "ReqQty",
                   "ParentItemCode", "SourceLineNum"
              FROM :lt_accum
            UNION ALL
            SELECT "ItemCode", "WhsCode", "ReqQty",
                   CAST("ParentItemCode" AS NVARCHAR(1000)),
                   TO_NVARCHAR("SourceLineNum")
              FROM :lt_wave;

        lt_sum =
            SELECT "ItemCode", "WhsCode", SUM("ReqQty") AS "ReqQty"
              FROM :lt_union
             GROUP BY "ItemCode", "WhsCode";

        lt_labels =
            SELECT "ItemCode", "WhsCode",
                   STRING_AGG("ParentItemCode", ', ') AS "ParentItemCode",
                   STRING_AGG("SourceLineNum", ', ')  AS "SourceLineNum"
              FROM (
                    SELECT DISTINCT "ItemCode", "WhsCode",
                           "ParentItemCode", "SourceLineNum"
                      FROM :lt_union
                   )
             GROUP BY "ItemCode", "WhsCode";

        lt_accum =
            SELECT S."ItemCode", S."WhsCode", S."ReqQty",
                   L."ParentItemCode", L."SourceLineNum"
              FROM :lt_sum S
              JOIN :lt_labels L
                ON L."ItemCode" = S."ItemCode" AND L."WhsCode" = S."WhsCode";

        -- b) отбираем те позиции, у которых текущая глубина = их LLC (то есть больше никаких дальнейших потребностей по ним не придёт)
        lt_due =
            SELECT A."ItemCode", A."WhsCode", A."ReqQty",
                   A."ParentItemCode", A."SourceLineNum"
              FROM :lt_accum A
              JOIN :lt_item_llc L
                ON L."ItemCode" = A."ItemCode" AND L."WhsCode" = A."WhsCode"
             WHERE L."LLC" = :lv_iter;

        -- c) достаём остатки, признак "производственный товар" и сразу считаем
        --    AvailableQty / ShortageQty (без GREATEST() - через CASE, чтобы не
        --    зависеть от наличия этой функции в конкретной ревизии HANA).
        --
        --    Для производственных товаров ShortageQty дополнительно
        --    округляется ВВЕРХ до кратного OITT."Qauntity" (размер партии
        --    по спецификации) - произвести можно только целое число партий,
        --    и раз нужно закрыть весь дефицит, округление именно вверх, а
        --    не до ближайшего (иначе может не хватить). Пример: дефицит 35
        --    при партии 16 -> CEIL(35/16)=3 партии -> рекомендация 48, а
        --    не 35 и не 32. "RequiredQty" в результате остаётся "чистой",
        --    несокращённой потребностью (35) - это для сверки; именно
        --    "ShortageQty" (48) - то, что реально стоит произвести, и именно
        --    оно используется дальше для расчёта потребности в компонентах.
        --
        --    Для ЗАКУПОЧНЫХ товаров (нет своей спецификации) ShortageQty
        --    точно так же округляется ВВЕРХ, но до кратного OITM."OrdrMulti"
        --    (кратность заказа у поставщика) - если это поле у товара
        --    заполнено (> 0). OrdrMulti используется ТОЛЬКО когда у товара
        --    нет своей спецификации (BT."Code" IS NULL) - для производственных
        --    товаров всегда действует Qauntity выше, независимо от того,
        --    заполнен ли у них OrdrMulti (см. примечание в начале файла:
        --    один и тот же полуфабрикат может входить в разные рецепты с
        --    разной кратностью, поэтому кратность должна идти со стороны
        --    спецификации, а не с товара). Если у закупочного товара
        --    OrdrMulti не заполнен - округления нет, как и раньше.
        lt_due_detail =
            SELECT
                D."ItemCode", D."WhsCode", D."ReqQty",
                D."ParentItemCode", D."SourceLineNum",
                I."ItemName",
                CASE WHEN BT."Code" IS NOT NULL THEN 'Y' ELSE 'N' END AS "IsManufactured",
                I."OrdrMulti" AS "OrderMulti",
                COALESCE(W."OnHand", 0)     AS "OnHandQty",
                COALESCE(W."IsCommited", 0) AS "CommittedQty",
                COALESCE(W."OnOrder", 0)    AS "OrderedQty",
                (COALESCE(W."OnHand", 0) - COALESCE(W."IsCommited", 0) + COALESCE(W."OnOrder", 0)) AS "AvailableQty",
                CASE
                    WHEN D."ReqQty" - (COALESCE(W."OnHand", 0) - COALESCE(W."IsCommited", 0) + COALESCE(W."OnOrder", 0)) <= 0
                        THEN 0
                    WHEN BT."Code" IS NOT NULL THEN
                        CEIL(
                            (D."ReqQty" - (COALESCE(W."OnHand", 0) - COALESCE(W."IsCommited", 0) + COALESCE(W."OnOrder", 0)))
                            / CASE WHEN COALESCE(BT."Qauntity", 0) <= 0 THEN 1 ELSE BT."Qauntity" END
                        )
                        * CASE WHEN COALESCE(BT."Qauntity", 0) <= 0 THEN 1 ELSE BT."Qauntity" END
                    WHEN COALESCE(I."OrdrMulti", 0) > 0 THEN
                        CEIL(
                            (D."ReqQty" - (COALESCE(W."OnHand", 0) - COALESCE(W."IsCommited", 0) + COALESCE(W."OnOrder", 0)))
                            / I."OrdrMulti"
                        ) * I."OrdrMulti"
                    ELSE
                        D."ReqQty" - (COALESCE(W."OnHand", 0) - COALESCE(W."IsCommited", 0) + COALESCE(W."OnOrder", 0))
                END AS "ShortageQty"
              FROM :lt_due D
              JOIN "OITM" I ON I."ItemCode" = D."ItemCode"
              LEFT JOIN "OITW" W
                ON W."ItemCode" = D."ItemCode" AND W."WhsCode" = D."WhsCode"
              -- MAX(Qauntity) защищает от задвоения строк/дефицита, если у
              -- товара окажется больше одной строки OITT с этим TreeType
              -- (например, из-за разных прайс-листов) - без этого JOIN
              -- размножил бы D на каждую такую строку.
              LEFT JOIN (
                    SELECT "Code", MAX("Qauntity") AS "Qauntity"
                      FROM "OITT"
                     WHERE "TreeType" = :lv_treetype
                     GROUP BY "Code"
                   ) BT
                ON BT."Code" = D."ItemCode";

        -- d) пишем в результат только позиции с реальным дефицитом
        --    (для полной видимости, в т.ч. позиций без дефицита, убеpите
        --    условие WHERE "ShortageQty" > 0 ниже)
        lt_out =
            SELECT * FROM :lt_out
            UNION ALL
            SELECT
                CASE WHEN "IsManufactured" = 'Y' THEN 2 ELSE 1 END AS "HierarchyLevel",
                CAST(NULL AS INTEGER) AS "ProductionSequence",
                :lv_iter AS "BomDepth",
                "ItemCode", "ItemName", "WhsCode",
                "ReqQty" AS "RequiredQty",
                "OnHandQty", "CommittedQty", "OrderedQty",
                "AvailableQty", "ShortageQty", "OrderMulti",
                "ParentItemCode", "SourceLineNum", "IsManufactured"
              FROM :lt_due_detail
             WHERE "ShortageQty" > 0;

        -- e) следующая волна: компоненты BoM только тех произв. позиций,
        --    у которых есть дефицит - именно на величину дефицита, а не
        --    полной потребности.
        --
        --    ВАЖНО: OITT."Qauntity" 
        --    это плановое количество родителя, которое получается за ОДИН
        --    прогон рецептуры ( пример: для 13532 "DESSERT HONEY CAKE" Qauntity = 16), а количества в ITT1."Quantity"
        --    заданы на этот целый батч, а не на 1 единицу родителя. Поэтому расход компонента на единицу родителя =
        --    ITT1."Quantity" / OITT."Qauntity", а не просто ITT1."Quantity". COALESCE/CASE защищает от деления на 0 или NULL (тогда считаем
        --    batch = 1).
        lt_wave =
            SELECT
                C."Code" AS "ItemCode",
                DD."WhsCode",
                (DD."ShortageQty" * C."Quantity"
                    / CASE WHEN COALESCE(T."Qauntity", 0) = 0 THEN 1 ELSE T."Qauntity" END
                ) AS "ReqQty",
                DD."ItemCode" AS "ParentItemCode",
                CAST(NULL AS INTEGER) AS "SourceLineNum"
              FROM :lt_due_detail DD
              -- MAX(Qauntity) - та же защита от задвоения, что и в шаге (c), на случай нескольких строк OITT под одним TreeType.
              JOIN (
                    SELECT "Code", MAX("Qauntity") AS "Qauntity"
                      FROM "OITT"
                     WHERE "TreeType" = :lv_treetype
                     GROUP BY "Code"
                   ) T ON T."Code" = DD."ItemCode"
              JOIN "ITT1" C ON C."Father" = T."Code"
             WHERE DD."IsManufactured" = 'Y'
               AND DD."ShortageQty" > 0;

        -- f) убираем обработанные позиции из аккумулятора
        lt_accum =
            SELECT A."ItemCode", A."WhsCode", A."ReqQty", A."ParentItemCode", A."SourceLineNum"
              FROM :lt_accum A
              LEFT JOIN :lt_due D
                ON D."ItemCode" = A."ItemCode" AND D."WhsCode" = A."WhsCode"
             WHERE D."ItemCode" IS NULL;

        lv_iter := lv_iter + 1;
    END WHILE;

    -- ------------------------------------------------------------------
    -- 2.4. Порядок создания производственных заказов: самый глубокий (самый "маленький") BoM - первый; верхний, финальный - последний.
    -- ------------------------------------------------------------------
    OT_RESULT =
        SELECT
            "HierarchyLevel",
            CASE WHEN "HierarchyLevel" = 2
                 THEN DENSE_RANK() OVER (PARTITION BY "HierarchyLevel" ORDER BY "BomDepth" DESC)
                 ELSE CAST(NULL AS INTEGER) END AS "ProductionSequence",
            "BomDepth", "ItemCode", "ItemName", "WhsCode",
            "RequiredQty", "OnHandQty", "CommittedQty", "OrderedQty",
            "AvailableQty", "ShortageQty", "OrderMulti",
            "ParentItemCode", "SourceLineNum", "IsManufactured"
          FROM :lt_out
         ORDER BY "HierarchyLevel", "ProductionSequence", "BomDepth" DESC, "ItemCode";

END;
