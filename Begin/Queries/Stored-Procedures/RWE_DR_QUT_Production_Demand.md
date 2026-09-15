CREATE PROCEDURE "RWE_DR_QUT_Production_Demand"
(IN DocNum INT)

LANGUAGE SQLSCRIPT 
SQL SECURITY INVOKER
AS  	
	
WhsCode NVARCHAR(8);		
DocEntry INT;
				
/*				
* Procedure version: 20260810.ASH				
* Report Name: Shortcut 			
* Creation Date: 10.08.2026				
* Creator: RoinWorld (Alex Shtyrov)				
* Requested by: Elena Kris		
* Report Description: Shows Demand for Sales BoM on specified in doc warehouse				
* Object type: 
* Category: Shortcut				
*				
* Change log: 	
* 		
* 10/08/2026 - Alex Shtyrov - Initial Version
*/	

BEGIN SEQUENTIAL EXECUTION	

DECLARE lt_result TABLE 
( "HierarchyLevel"     NVARCHAR(15),        -- 1 = закупить, 2 = произвести
  "ProductionSequence" INTEGER,        -- порядок создания произв. заказов (ASC), NULL для уровня 1
  "BomDepth"           INTEGER,        -- уровень вложенности BoM, на котором позиция была "закрыта" (0 = сама строка Quotation)
  "ItemCode"           NVARCHAR(50),
  "ItemName"           NVARCHAR(200),
  "WhsCode"            NVARCHAR(8),
  "RequiredQty"        DECIMAL(19,6),  -- суммарная потребность, агрегированная по всем родителям
  "OnHandQty"          DECIMAL(19,6),
  "CommittedQty"       DECIMAL(19,6),
  "OrderedQty"         DECIMAL(19,6),
  "AvailableQty"       DECIMAL(19,6),  -- OnHand - Committed + Ordered
  "ShortageQty"        DECIMAL(19,6),  -- рекомендуемое к закупке / производству количество
  "OrderMulti"         DECIMAL(19,6),  -- OITM."OrdrMulti" товара (для справки; реально применяется к округлению ShortageQty только если IsManufactured = 'N')
  "ParentItemCode"     NVARCHAR(1000), -- родитель(и), сгенерировавшие потребность (список, если их несколько)
  "SourceLineNum"      NVARCHAR(500),  -- строка(и)-источник в QUT1 (список)
  "IsManufactured"     NVARCHAR(1)     -- 'Y'/'N', диагностика
);

SELECT R."DocEntry" INTO DocEntry FROM NNM1 N
INNER JOIN OQUT R ON N."ObjectCode" = R."ObjType" AND N."Series" = R."Series"
WHERE R."DocNum" = :DocNum;

CALL "RWE_PROC_QUT_DEMAND"(4,:lt_result);

SELECT CASE WHEN "HierarchyLevel" = 1 THEN 'Закупка'
			WHEN "HierarchyLevel" = 2 THEN 'Производство'
			END AS "Тип"
,"ProductionSequence" AS "Очерёдность"
,"BomDepth" AS "Глубина"
,"ItemCode" AS "Код Товара"
,"ItemName" AS "Наименование товара"
,"WhsCode" AS "Код Склада"
,"RequiredQty" AS "Требуемое к-во"
,"ShortageQty" AS "Кол-во для заказа"
,"OrderMulti" AS "Кратность заказа"
,"OnHandQty" AS "На складе"
,"CommittedQty" AS "Заказано (продажа)"
,"OrderedQty" AS "Заказано (закупка)"
,"ParentItemCode" AS "Требуется в Товарах"
--,"SourceLineNum" AS "Строка док-та"
FROM :lt_result;

END
