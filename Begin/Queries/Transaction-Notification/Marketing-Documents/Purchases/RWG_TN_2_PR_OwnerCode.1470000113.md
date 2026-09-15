CREATE PROCEDURE "RWG_TN_2_PR_OwnerCode.1470000113"
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
WHERE PR."DocEntry" = :objKey
AND IFNULL(PR."OwnerCode",-1) = -1;
	
	
IF :DocEntry=-1 THEN RETURN; END IF;
	
error := N'Подпись Ответственного является обязательной'; 
	
END;
