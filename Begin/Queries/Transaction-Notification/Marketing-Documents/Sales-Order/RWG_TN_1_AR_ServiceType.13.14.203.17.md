CREATE PROCEDURE "RWG_TN_1_AR_ServiceType.13.14.203.17"
(IN objType  NVARCHAR ( 30),				
IN  objKey   NVARCHAR (255),				
IN  action   NVARCHAR ( 1),	
--IN  user_id  INT,			
OUT error    NVARCHAR (200))	
			
LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	 

/*
* Procedure version: 20260907.SAY
* Report Name: TN
* Creation Date: 07.09.2026
* Creator: SAY (Alex Shtyrov)
* Requested by: 
* Category: DataValidation
* Change log: 
* 07.09.2026 - Initial version
*/

BEGIN SEQUENTIAL EXECUTION 

DECLARE DocEntry INT;
DECLARE tabName  NVARCHAR(20);
DECLARE sqlStmt  NVARCHAR(5000);
	
IF :action NOT IN ('A','U') THEN RETURN; END IF;

tabName :=
    CASE :objType
    WHEN '13'        THEN 'INV'
    WHEN '14'        THEN 'RIN'
    WHEN '203'       THEN 'DPI'
    WHEN '17'        THEN 'RDR'
    ELSE 'RETURN'
    END;
IF :tabName ='RETURN' THEN RETURN;
END IF;

sqlStmt := 'SELECT IFNULL(MAX(P1."DocEntry"), -1) ' ||
		   'FROM  O' || :tabName || ' P ' ||
           'WHERE P."DocEntry" = ? AND P1."DocType" =''S'' '
           ;
 
EXECUTE IMMEDIATE :sqlStmt INTO DocEntry USING :objKey;
 
IF :DocEntry = -1 THEN RETURN;
END IF;

error := N'Документы Сервисного типа запрещены, используйте S-Item"';
		

END;
