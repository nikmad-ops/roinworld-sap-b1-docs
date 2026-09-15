CREATE PROCEDURE "RWG_TN_1_CheckBP_TaxID.2."
(IN objType  NVARCHAR ( 30),				
IN  objKey   NVARCHAR (255),				
IN  action   NVARCHAR ( 1),	
--IN  user_id  INT,			
OUT error    NVARCHAR (200))	
			
LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	 

/*
* Procedure version: 20260814.SAY
* Report Name: TN
* Creation Date: 29.01.2026
* Creator: SAY (Alex Shtyrov)
* Requested by: 
* Category: DataValidation
* Change log: 
* 14.08.2026 - Initial version
*/

BEGIN SEQUENTIAL EXECUTION 

DECLARE DocEntry INT;
	
IF :action NOT IN ('A','U') THEN RETURN; END IF;
	
IF :objType='2' THEN
	SELECT IFNULL(MAX(T0."DocEntry"),-1) INTO DocEntry
	FROM OCRD T0 
	WHERE T0."CardCode" = :objKey
	AND IFNULL(T0."LicTradNum",'') = ''
	;
	
END IF;
	
IF :DocEntry=-1 THEN RETURN; END IF;
	
error := N'Поле Federal Tax ID является обязательным'; 
	
END;

