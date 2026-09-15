CREATE PROCEDURE "RWG_TN_1_WriteSpec_.60."
(IN objType  NVARCHAR ( 30),				
IN  objKey   NVARCHAR (255),				
IN  action   NVARCHAR ( 1),	
--IN  user_id  INT,			
OUT error    NVARCHAR (200))	
			
LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	 

/*
* Procedure version: 20260908.SAY
* Report Name: TN
* Creation Date: 08.09.2026
* Creator: SAY (Alex Shtyrov)
* Requested by: 
* Category: DataValidation
* Change log: 
* 08.09.2026 - Initial version
*/

BEGIN SEQUENTIAL EXECUTION 

DECLARE DocEntry INT;
	
IF :action NOT IN ('A','U') THEN RETURN; END IF;
	
SELECT IFNULL(MAX("DocEntry"),-1) INTO DocEntry
FROM OIGE
WHERE "RelatedTyp" <> '59' AND "DocEntry" = :objKey
AND IFNULL("U_RWG_WriteOffSpec",'') = ''
;
	
IF :DocEntry=-1 THEN RETURN; END IF;
	
error := N'Не заполнено поле Причина списания'; 
	
END;
