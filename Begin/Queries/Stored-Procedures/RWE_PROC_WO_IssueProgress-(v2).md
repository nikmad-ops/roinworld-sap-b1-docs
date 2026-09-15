CREATE PROCEDURE "RWE_PROC_WO_IssueProgress" (
    IN  IV_PARENT_DOCENTRY INTEGER,  -- DocEntry родительского OWOR; NULL = вся система
    OUT OT_RESULT TABLE (
        "ParentProdOrderDocEntry" INTEGER,
        "ParentProdOrderDocNum"   INTEGER,
        "ComponentLineNum"        INTEGER,
        "ComponentItemCode"       NVARCHAR(50),
        "ComponentItemName"       NVARCHAR(200),
        "ComponentPlannedQty"     DECIMAL(19,6), -- WOR1."PlannedQty" родителя - сколько нужно по спецификации
        "ComponentWhsCode"        NVARCHAR(8),   -- склад по спецификации у родителя (WOR1."wareHouse")
        "ChildProdOrderDocEntry"  INTEGER,
        "ChildProdOrderDocNum"    INTEGER,
        "ChildProdOrderStatusRaw" NVARCHAR(1),   -- 'P'/'R'/'L'
        "ReceivedQty"             DECIMAL(19,6), -- SUM(IGN1.Quantity) по дочернему заказу
        "IssuedQty"               DECIMAL(19,6), -- SUM(IGE1.Quantity) по строке компонента родителя (только привязанные к произв.заказу документы)
        "GapQty"                  DECIMAL(19,6), -- ReceivedQty - IssuedQty
        "OnHandAtWhs"             DECIMAL(19,6), -- текущий физический остаток на складе спецификации (OITW."OnHand") - см. примечание в шапке файла
        "OtherGoodsIssueQty"      DECIMAL(19,6), -- справочно: списано др. документами "Выбытие со склада", не привязанными к произв.заказу (см. примечание в шапке файла)
        "Status"                  NVARCHAR(60)
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS
BEGIN
    DECLARE lv_prodorder_obj INTEGER := 202; -- код объекта "Производственный заказ" (подтверждено)

    lt_links =
        SELECT
            W."DocEntry"     AS "ParentProdOrderDocEntry",
            PO."DocNum"      AS "ParentProdOrderDocNum",
            W."LineNum"      AS "ComponentLineNum",
            W."ItemCode"     AS "ComponentItemCode",
            IM."ItemName"    AS "ComponentItemName",
            W."PlannedQty"   AS "ComponentPlannedQty",
            W."wareHouse"    AS "ComponentWhsCode",
            W."PoDocEntry"   AS "ChildProdOrderDocEntry",
            W."PoDocNum"     AS "ChildProdOrderDocNum",
            CO."Status"      AS "ChildProdOrderStatusRaw"
          FROM "WOR1" W
          JOIN "OWOR" PO ON PO."DocEntry" = W."DocEntry"
          JOIN "OWOR" CO ON CO."DocEntry" = W."PoDocEntry"
          JOIN "OITM" IM ON IM."ItemCode" = W."ItemCode"
         WHERE W."PoDocType" = :lv_prodorder_obj
           AND W."PoDocEntry" IS NOT NULL
           AND (:iv_parent_docentry IS NULL OR W."DocEntry" = :iv_parent_docentry);

    -- сколько полуфабриката реально оприходовано по дочернему заказу
    -- (BaseEntry = DocEntry дочернего заказа)
    lt_received =
        SELECT I."BaseEntry" AS "ChildProdOrderDocEntry", SUM(I."Quantity") AS "ReceivedQty"
          FROM "IGN1" I
          JOIN "OIGN" H ON H."DocEntry" = I."DocEntry"
         WHERE I."BaseType" = :lv_prodorder_obj
           AND H."CANCELED" = 'N'
         GROUP BY I."BaseEntry";

    -- сколько полуфабриката реально списано именно в РОДИТЕЛЬСКИЙ заказ
    -- (BaseEntry = DocEntry родителя, BaseLine = LineNum строки компонента)
    lt_issued =
        SELECT I."BaseEntry" AS "ParentProdOrderDocEntry", I."BaseLine" AS "ComponentLineNum",
               SUM(I."Quantity") AS "IssuedQty"
          FROM "IGE1" I
          JOIN "OIGE" H ON H."DocEntry" = I."DocEntry"
         WHERE I."BaseType" = :lv_prodorder_obj
           AND H."CANCELED" = 'N'
         GROUP BY I."BaseEntry", I."BaseLine";

    -- справочно: списания этого товара документами "Выбытие со склада",
    -- НЕ привязанными к производственному заказу (BaseType <> 202 или
    -- пусто) - см. примечание "b)" в шапке файла (не фильтруется по
    -- складу - колонка склада в IGE1 пока не подтверждена).
    lt_otherissue =
        SELECT I."ItemCode", SUM(I."Quantity") AS "OtherGoodsIssueQty"
          FROM "IGE1" I
          JOIN "OIGE" H ON H."DocEntry" = I."DocEntry"
         WHERE (I."BaseType" IS NULL OR I."BaseType" <> :lv_prodorder_obj)
           AND H."CANCELED" = 'N'
         GROUP BY I."ItemCode";

    -- текущий физический остаток на складе, который требуется у родителя
    -- по спецификации (ComponentWhsCode) - используется для проверки
    -- "на самом деле уже списано" (см. правило статуса в шапке файла)
    lt_onhand =
        SELECT "ItemCode", "WhsCode", "OnHand"
          FROM "OITW";

    OT_RESULT =
        SELECT
            L."ParentProdOrderDocEntry",
            L."ParentProdOrderDocNum",
            L."ComponentLineNum",
            L."ComponentItemCode",
            L."ComponentItemName",
            L."ComponentPlannedQty",
            L."ComponentWhsCode",
            L."ChildProdOrderDocEntry",
            L."ChildProdOrderDocNum",
            L."ChildProdOrderStatusRaw",
            COALESCE(RQ."ReceivedQty", 0) AS "ReceivedQty",
            COALESCE(IQ."IssuedQty", 0)   AS "IssuedQty",
            COALESCE(RQ."ReceivedQty", 0) - COALESCE(IQ."IssuedQty", 0) AS "GapQty",
            COALESCE(OH."OnHand", 0)      AS "OnHandAtWhs",
            COALESCE(OI."OtherGoodsIssueQty", 0) AS "OtherGoodsIssueQty",
            CASE
                WHEN COALESCE(RQ."ReceivedQty", 0) = 0
                    THEN 'Полуфабрикат ещё не оприходован'
                WHEN COALESCE(IQ."IssuedQty", 0) = 0
                    THEN 'Списание не выполнено'
                WHEN COALESCE(IQ."IssuedQty", 0) < COALESCE(RQ."ReceivedQty", 0)
                    THEN
                        -- строго в этой последовательности: проверка остатка
                        -- на складе добавлена ТОЛЬКО здесь, внутри ветки
                        -- "частичное" - см. примечание в шапке файла
                        CASE
                            WHEN COALESCE(OH."OnHand", 0) = 0
                                THEN 'Списано полностью (Goods Issue)'
                            ELSE 'Списание частичное, остаток ' || TO_NVARCHAR(COALESCE(RQ."ReceivedQty", 0) - COALESCE(IQ."IssuedQty", 0))
                        END
                ELSE 'Списано полностью'
            END AS "Status"
          FROM :lt_links L
          LEFT JOIN :lt_received  RQ ON RQ."ChildProdOrderDocEntry" = L."ChildProdOrderDocEntry"
          LEFT JOIN :lt_issued    IQ ON IQ."ParentProdOrderDocEntry" = L."ParentProdOrderDocEntry"
                                    AND IQ."ComponentLineNum" = L."ComponentLineNum"
          LEFT JOIN :lt_onhand    OH ON OH."ItemCode" = L."ComponentItemCode"
                                    AND OH."WhsCode" = L."ComponentWhsCode"
          LEFT JOIN :lt_otherissue OI ON OI."ItemCode" = L."ComponentItemCode"
         ORDER BY L."ParentProdOrderDocEntry", L."ComponentLineNum";

END;
