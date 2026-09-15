CREATE PROCEDURE "RWG_TN_2_WhsCode.17"
(IN objType  NVARCHAR ( 30),				
IN  objKey   NVARCHAR (255),				
IN  action   NVARCHAR ( 1),	
--IN  user_id  INT,			
OUT error    NVARCHAR (200))	
			
LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	 

/*
* Procedure version: 20260826.SAY
* Report Name: TN
* Creation Date: 26.08.2026
* Creator: SAY (Alex Shtyrov)
* Requested by: 
* Category: DataValidation
* Change log: 
* 26.08.2026 - Initial version
*/

BEGIN SEQUENTIAL EXECUTION 

DECLARE CountWhs INT = 0;
	
IF :action NOT IN ('A','U') THEN RETURN; END IF;
	
CREATE LOCAL TEMPORARY TABLE #Whs
("WhsCode" NVARCHAR(8));

-- Проверить сколько разных складов укзаано в строках (может быть только 1)
INSERT INTO #Whs
	SELECT R."WhsCode"
	FROM RDR1 R
	WHERE R."DocEntry" = :objKey
	GROUP BY R."WhsCode";

SELECT COUNT("WhsCode") INTO CountWhs FROM #Whs;

DROP TABLE #Whs;
	
-- Формирование текста ошибок
IF :CountWhs >1 THEN 
error := N'Склад должен быть единым для всех строк документа продаж'; 
END IF;
	
END;
