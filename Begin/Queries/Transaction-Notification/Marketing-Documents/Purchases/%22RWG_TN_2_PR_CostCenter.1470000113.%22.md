CREATE PROCEDURE "RWG_TN_2_PR_CostCenter.1470000113."
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
	
IF :action NOT IN ('A','U') THEN RETURN; END IF;
		
SELECT IFNULL(MAX(PR."DocEntry"),-1) INTO DocEntry
FROM OPRQ PR
INNER JOIN PRQ1 PR1 ON PR."DocEntry" = PR1."DocEntry"
WHERE PR."DocEntry" = :objKey
AND IFNULL(PR1."OcrCode",'N') = 'N' OR IFNULL(PR1."OcrCode2",'N') = 'N'
;
		
IF :DocEntry=-1 THEN RETURN; END IF;
	
error := N'Поля Key/Milestone и Department являются обязательными'; 
	
END;
