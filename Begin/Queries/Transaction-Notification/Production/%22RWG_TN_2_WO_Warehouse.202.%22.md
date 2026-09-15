ALTER PROCEDURE "RWG_TN_2_WO_Warehouse.202."
(IN objType  NVARCHAR ( 30),				
IN  objKey   NVARCHAR (255),				
IN  action   NVARCHAR ( 1),	
--IN  user_id  INT,			
OUT error    NVARCHAR (200))	
			
LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	 

/*
* Procedure version: 20260824.SAY
* Report Name: TN
* Creation Date: 24.08.2026
* Creator: SAY (Alex Shtyrov)
* Requested by: 
* Category: DataValidation
* Change log: 
* 24.08.2026 - Initial version
*/

BEGIN SEQUENTIAL EXECUTION 

DECLARE DocEntry INT;
	
IF :action NOT IN ('U') THEN RETURN; END IF;
		
SELECT IFNULL(MAX(W."DocEntry"),-1) INTO DocEntry
FROM OWOR O
INNER JOIN WOR1 W ON O."DocEntry" = W."DocEntry"
WHERE O."DocEntry" = :objKey 
AND O."Warehouse" <> W."wareHouse"
;
	
IF :DocEntry=-1 THEN RETURN; END IF;
	
error := N'Склад продукта не соответствует складу компонента'; 
	
END;
