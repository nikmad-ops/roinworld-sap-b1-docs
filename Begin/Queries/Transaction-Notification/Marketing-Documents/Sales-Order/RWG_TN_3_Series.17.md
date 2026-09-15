ALTER PROCEDURE "RWG_TN_3_Series.17"
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

DECLARE WhsCode NVARCHAR(8);
DECLARE NumWhs NVARCHAR(8);
DECLARE DocEntry INT;

IF :action NOT IN ('A','U') THEN RETURN; END IF;

CREATE LOCAL TEMPORARY TABLE #WhsList
("WhsCode" NVARCHAR(8));
	
SELECT MAX("WhsCode") INTO WhsCode 
FROM RDR1
WHERE "DocEntry" = :objKey;

SELECT "U_RWE_RefWhsCode" INTO NumWhs
FROM ORDR O
INNER JOIN NNM1 N 
ON O."ObjType" = N."ObjectCode" 
AND O."Series" = N."Series"
WHERE O."DocEntry" = :objKey;

	 
IF :NumWhs IN ('-','All') THEN
	INSERT INTO #WhsList
		SELECT 'GN' FROM DUMMY
		UNION ALL
		SELECT 'ST' FROM DUMMY
		UNION ALL
		SELECT 'FT' FROM DUMMY
		UNION ALL 
		SELECT 'BK' FROM DUMMY;
END IF;

IF :NumWhs IN ('ST') THEN
	INSERT INTO #WhsList
		SELECT 'ST' FROM DUMMY;
END IF;

IF :NumWhs IN ('FT') THEN
	INSERT INTO #WhsList
		SELECT 'FT' FROM DUMMY;
END IF;

IF :NumWhs IN ('BK') THEN
	INSERT INTO #WhsList
		SELECT 'BK' FROM DUMMY;
END IF;

SELECT IFNULL(MAX(R."DocEntry"),-1) INTO DocEntry
FROM RDR1 R
WHERE R."WhsCode" IN (SELECT "WhsCode" FROM #WhsList)
AND R."DocEntry" = :objKey
;

DROP TABLE #WhsList;

-- Формирование текста ошибок
IF :DocEntry = -1 THEN 
error := N'Серия Нумерации не соответствует указанному Складу'; 
END IF;
	
END;
