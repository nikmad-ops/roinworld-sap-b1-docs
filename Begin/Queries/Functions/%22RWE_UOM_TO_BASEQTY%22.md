CREATE FUNCTION "RWE_UOM_TO_BASEQTY" (
    IN IV_ITEMCODE NVARCHAR(50),
    IN IV_UOMENTRY INTEGER,
    IN IV_QTY      DECIMAL(19,6)
)
RETURNS RV_QTY DECIMAL(19,6)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
READS SQL DATA AS

/* Function version 20260824.SAY
*  Функция перерасчёта количества в базовую ЕИ
*/

BEGIN
    DECLARE lv_ugpentry  INTEGER;
    DECLARE lv_baseuom   INTEGER;
    DECLARE lv_basefctr  DECIMAL(19,6);
    DECLARE lv_altfctr   DECIMAL(19,6);

    IF :iv_qty IS NULL THEN
        rv_qty := 0;
    ELSEIF :iv_uomentry IS NULL THEN
        rv_qty := :iv_qty;
    ELSE
        SELECT MAX("UgpEntry") INTO lv_ugpentry
          FROM "OITM"
         WHERE "ItemCode" = :iv_itemcode;

        IF lv_ugpentry IS NULL THEN
            -- товар не привязан к группе UoM - считаем, что количество уже дано в базовой (складской) единице измерения
            rv_qty := :iv_qty;
        ELSE
            SELECT MAX("BaseUom") INTO lv_baseuom
              FROM "OUGP"
             WHERE "UgpEntry" = lv_ugpentry;

            IF lv_baseuom IS NOT NULL AND lv_baseuom = :iv_uomentry THEN
                -- строка уже в базовой единице группы
                rv_qty := :iv_qty;
            ELSE
                SELECT MAX("BaseQty"), MAX("AltQty") INTO lv_basefctr, lv_altfctr
                  FROM "UGP1"
                 WHERE "UgpEntry" = lv_ugpentry
                   AND "UomEntry" = :iv_uomentry;

                IF lv_basefctr IS NULL OR lv_altfctr IS NULL OR lv_altfctr = 0 THEN
                    -- нет строки конверсии для этой единицы - fail-safe: без пересчёта
                    rv_qty := :iv_qty;
                ELSE
                    rv_qty := :iv_qty * (lv_basefctr / lv_altfctr);
                END IF;
            END IF;
        END IF;
    END IF;
END;
