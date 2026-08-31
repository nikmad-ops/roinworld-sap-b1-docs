ALTER PROCEDURE "RWG_TN_1_AP_NumAtCard.18.19"
(IN objType  NVARCHAR ( 30),				
IN  objKey   NVARCHAR (255),				
IN  action   NVARCHAR ( 1),	
--IN  user_id  INT,			
OUT error    NVARCHAR (200))	
			
LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	 

/*
* Procedure version: 20260831.SAY
* Report Name: TN
* Creation Date: 31.08.2026
* Creator: SAY (Alex Shtyrov)
* Requested by: 
* Category: DataValidation
* Change log: 
* 31.08.2026 - Initial version
*/

BEGIN SEQUENTIAL EXECUTION 

DECLARE DocEntry INT;
DECLARE tabName  NVARCHAR(20);
DECLARE sqlStmt  NVARCHAR(5000);
	
IF :action NOT IN ('A','U') THEN RETURN; END IF;

tabName :=
    CASE :objType
    WHEN '18'        THEN 'PCH'
    WHEN '19'        THEN 'RPC'
    ELSE 'RETURN'
    END;
IF :tabName ='RETURN' THEN RETURN;
END IF;

sqlStmt := 'SELECT IFNULL(MAX(P."DocEntry"), -1) ' ||
		   'FROM  O' || :tabName || ' P ' ||
           'WHERE P."DocEntry" = ? AND P."NumAtCard" IS NULL '
           ;
 
EXECUTE IMMEDIATE :sqlStmt INTO DocEntry USING :objKey;
 
IF :DocEntry = -1 THEN RETURN;
END IF;

error := N'Поле № Документа БП обязательно к заполнению';
		

END;