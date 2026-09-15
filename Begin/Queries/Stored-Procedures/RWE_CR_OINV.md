CREATE PROCEDURE "RWE_CR_OINV"
(IN DocKey INT,
 IN ObjectId INT)

LANGUAGE SQLSCRIPT 
SQL SECURITY DEFINER
AS  	

/*				
* Procedure version: 20260826.ASH				
* Report Name: Crystal Report			
* Creation Date: 26.08.2026				
* Creator: RoinWorld (Alex Shtyrov)				
* Requested by: 
* Report Description: Procedure for Crystal Report on AR Invoice		
* Object type: 
* Category: CR				
*				
* Change log: 	
* 		
* 26/08/2026 - Alex Shtyrov - Initial Version
*/	

BEGIN SEQUENTIAL EXECUTION	

CREATE LOCAL TEMPORARY TABLE #Adm
("CompName" NVARCHAR(100)
,"CompPhone" NVARCHAR(50)
,"CompEmail" NVARCHAR(50)
,"CompTaxId" NVARCHAR(30)
,"CompCountry" NVARCHAR(50)
,"CompCountryF" NVARCHAR(50)
,"CompZipCode" NVARCHAR(20)
,"CompZipCodeF" NVARCHAR(20)
,"CompCounty" NVARCHAR(50)
,"CompCountyF" NVARCHAR(50)
,"CompCity" NVARCHAR(15)
,"CompCityF" NVARCHAR(15)
,"CompStreet" NVARCHAR(50)
,"CompStreetF" NVARCHAR(50)
);

INSERT INTO #Adm ("CompName","CompPhone","CompEmail","CompTaxId","CompCountry","CompCountryF","CompZipCode","CompZipCodeF"
				 ,"CompCounty","CompCountyF","CompCity","CompCityF","CompStreet","CompStreetF")
 	SELECT LEFT(A."AliasName",100),A."Phone1",A."E_Mail",A."TaxIdNum",'Egypt' ,'مصر'   --A."Country"	A."Country"
 	,A1."ZipCode",A1."ZipCodeF",A1."County",A1."CountyF",A1."City",A1."CityF",A1."Street",A1."StreetF"
 	FROM OADM A
 	LEFT JOIN ADM1 A1 ON 1=1
 	;
 
-- SELECT * FROM #Adm; 

CREATE LOCAL TEMPORARY TABLE #Layout
("DocEntry" INT
,"DocNum" INT
,"DocDate" DATE
,"ObjType" INT
,"CardCode" NVARCHAR(30)
,"CANCELED" NVARCHAR(1)
,"NumAtCard" NVARCHAR(100)
,"VisOrder" INT
,"CodeBars" NVARCHAR(30)
,"ItemCode" NVARCHAR(30)
,"ItemName" NVARCHAR(150)
,"ItemNameF" NVARCHAR(150)
,"Quantity" DECIMAL(16,8)
,"NumPerMsr" DECIMAL(16,8)
,"UomCode" NVARCHAR(15)
,"UoMName" NVARCHAR(20)
,"UomCodeF" NVARCHAR(15)
,"CardName" NVARCHAR(100)
,"CardNameF" NVARCHAR(100) 
,"TaxId" NVARCHAR(30)
,"BPLangCode" NVARCHAR(30)
,"UseArabic" NVARCHAR(1)
,"UseEnglish" NVARCHAR(1)
,"UseRussian" NVARCHAR(1)
); 
 
INSERT INTO #Layout ("DocEntry","DocNum","DocDate","ObjType","CardCode","CardName","CardNameF","CANCELED","NumAtCard","VisOrder","CodeBars","ItemCode","ItemName"
			  		,"ItemNameF","Quantity","NumPerMsr","UomCode","UoMName","UomCodeF","TaxId","BPLangCode","UseArabic","UseEnglish","UseRussian") 
 	SELECT 
-- DocHeader
	 O."DocEntry"
	,O."DocNum"
	,O."DocDate"
	,O."ObjType"
	,O."CardCode"
	,BP."CardName"
	,BP."CardFName"
	,O."CANCELED"
	,O."NumAtCard"
--- DocLines
	,P1."VisOrder"
	,P1."CodeBars"
	,P1."ItemCode"
	,P1."Dscription"
	,P1."U_RWG_ItemNameLL"
	,P1."Quantity"
	,P1."NumPerMsr"
	,P1."UomCode"
	,P1."unitMsr"
	,P1."U_RWG_UoMNameLL"
-- BP
	,BP."LicTradNum" 
	,BP."LangCode" 	
	,BP."QryGroup1" 	-- Arabic
	,BP."QryGroup2" 	-- English
	,BP."QryGroup3" 	-- Russian
-- Item
	FROM OINV O
	INNER JOIN INV1 P1 ON O."DocEntry" = P1."DocEntry"
	INNER JOIN OCRD BP ON BP."CardCode" = O."CardCode"
	INNER JOIN OITM IT ON IT."ItemCode" = P1."ItemCode"
	WHERE O."DocEntry" = :DocKey
	;

SELECT "DocEntry","DocNum","DocDate","ObjType","CardCode","CardName","CardNameF","CANCELED","NumAtCard","VisOrder","CodeBars","ItemCode","ItemName"
			  		,"ItemNameF","Quantity","NumPerMsr","UomCode","UoMName","UomCodeF","TaxId","BPLangCode","UseArabic","UseEnglish","UseRussian"
	  ,"CompName","CompPhone","CompEmail","CompTaxId","CompCountry","CompCountryF","CompZipCode","CompZipCodeF","CompCounty","CompCountyF","CompCity"
	  ,"CompCityF","CompStreet","CompStreetF"
FROM #Layout 
INNER JOIN #Adm ON 1=1
;

DROP TABLE #Adm;
DROP TABLE #Layout;

END
