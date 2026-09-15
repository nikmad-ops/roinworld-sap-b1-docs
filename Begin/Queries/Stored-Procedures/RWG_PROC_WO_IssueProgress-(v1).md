CREATE PROCEDURE "RWG_PROC_WO_IssueProgress" (
    IN  IV_PARENT_DOCENTRY INTEGER,  -- DocEntry родительского OWOR; NULL = вся система
    OUT OT_RESULT TABLE (
        "ParentProdOrderDocEntry" INTEGER,
        "ParentProdOrderDocNum"   INTEGER,
        "ComponentLineNum"        INTEGER,
        "ComponentItemCode"       NVARCHAR(50),
        "ComponentItemName"       NVARCHAR(200),
        "ComponentPlannedQty"     DECIMAL(19,6), -- WOR1."PlannedQty" родителя - сколько нужно по спецификации
        "ChildProdOrderDocEntry"  INTEGER,
        "ChildProdOrderDocNum"    INTEGER,
        "ChildProdOrderStatusRaw" NVARCHAR(1),   -- 'P'/'R'/'L'
        "ReceivedQty"             DECIMAL(19,6), -- SUM(IGN1.Quantity) по дочернему заказу
        "IssuedQty"               DECIMAL(19,6), -- SUM(IGE1.Quantity) по строке компонента родителя
        "GapQty"                  DECIMAL(19,6), -- ReceivedQty - IssuedQty
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

    OT_RESULT =
        SELECT
            L."ParentProdOrderDocEntry",
            L."ParentProdOrderDocNum",
            L."ComponentLineNum",
            L."ComponentItemCode",
            L."ComponentItemName",
            L."ComponentPlannedQty",
            L."ChildProdOrderDocEntry",
            L."ChildProdOrderDocNum",
            L."ChildProdOrderStatusRaw",
            COALESCE(RQ."ReceivedQty", 0) AS "ReceivedQty",
            COALESCE(IQ."IssuedQty", 0)   AS "IssuedQty",
            COALESCE(RQ."ReceivedQty", 0) - COALESCE(IQ."IssuedQty", 0) AS "GapQty",
            CASE
                WHEN COALESCE(RQ."ReceivedQty", 0) = 0
                    THEN 'Полуфабрикат ещё не оприходован'
                WHEN COALESCE(IQ."IssuedQty", 0) = 0
                    THEN 'Списание не выполнено'
                WHEN COALESCE(IQ."IssuedQty", 0) < COALESCE(RQ."ReceivedQty", 0)
                    THEN 'Списание частичное, остаток ' || TO_NVARCHAR(COALESCE(RQ."ReceivedQty", 0) - COALESCE(IQ."IssuedQty", 0))
                ELSE 'Списано полностью'
            END AS "Status"
          FROM :lt_links L
          LEFT JOIN :lt_received RQ ON RQ."ChildProdOrderDocEntry" = L."ChildProdOrderDocEntry"
          LEFT JOIN :lt_issued   IQ ON IQ."ParentProdOrderDocEntry" = L."ParentProdOrderDocEntry"
                                   AND IQ."ComponentLineNum" = L."ComponentLineNum"
         ORDER BY L."ParentProdOrderDocEntry", L."ComponentLineNum";

END;
