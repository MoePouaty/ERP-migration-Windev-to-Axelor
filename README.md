# Migration des données de Windev vers l’ERP Axelor

## Contexte du projet
Ce projet s’inscrit dans le cadre d’une **migration de données ERP**.  
L’objectif est de **récupérer, transformer et préparer** les données issues d’un ancien système développé sous **Windev / HFSQL (.FIC)** afin de les rendre compatibles avec **l’ERP Axelor**, basé sur **PostgreSQL**.

Le cas pratique porte sur la **table des articles (produits)**, extraite du système source et à intégrer dans la table `base_product` d’Axelor.


---

## Consignes 
- Extraire les données de la table **ARTICLE.FIC** (Windev / HFSQL)
- **Filtrer les articles :**
  - Date de création **postérieure au 01/01/2017**
  - Exclure les familles **AAA** et **SOS**

- **Créer ou transformer les colonnes suivantes selon les instructions :**
- **dtype productTypeSelect** → Si TENUESTOCK = 1 → "Article" ; Si = 2 → "Prestation" ; Si = 3 → "Consignataire"  
- **productFamily.id** → Correspond à ARCLEUNIK (max 19 caractères)
- **created_on / startDate** → Valeur = date de création de l’article (`datecre`)
- **allow_to_force_purchase_qty** → Valeur fixe = false
- **allow_to_force_sale_qty** → Valeur fixe = false
- **code** → Reprendre le code article (`CODEART`)
- **description** → Reprendre `DESIGN`
- **full_name** → Concaténer `CODEART` + " " + `LIBART`
- **internal_description** → Reprendre `DESIGN` (texte)
- **name** → Reprendre `LIBART`
- **procurement_method_select** → Selon la famille (`CODEFAM`) : croiser avec l’onglet APPRO pour obtenir la valeur (“Acheter”, “Produire” ou “Acheter et Produire”)
- **start_date** → Même valeur que la date de création (`DATECRE`)
- **stock_managed** → Si TENUESTOCK = 1 → TRUE ; Si = 2 → FALSE
- **product_category** → Croiser `CODEFAM` avec l’onglet APPRO, colonne *Catégorie*
- **product_family** → Croiser `CODEFAM` avec l’onglet APPRO, colonne *Comptable*
- **purchase_currency** → Reprendre `DEVISE` ; si vide → “EUR”
- **unit** → Reprendre `UNITE` ; par défaut = “UNI” ou “M”

- **Préparer un fichier XML final** contenant les données transformées pour import dans **Axelor ERP**


---

## Fichiers à ma disposition
- Table **ARTICLE.FIC**
- Table **Consignes**
  <img width="1734" height="700" alt="Table consignes" src="https://github.com/user-attachments/assets/aa20e538-6c71-43f1-a6f6-23321e94947f" />

- Table **Appro**
  <img width="659" height="559" alt="Table appro" src="https://github.com/user-attachments/assets/e42b1edf-8986-49f5-9263-51c8ed66ef84" />

---

## Etape 1 : Filtrer via requette SQL

J'ai tout d'abord réalisé la requêtte SQL dans le **Centre de Contrôl HFSQL** pour filtrer la date et exclure les familles **AAA** et **SOS**
<img width="1754" height="939" alt="Requette SQL dans centre de contrôle HFSQL" src="https://github.com/user-attachments/assets/81772687-aa16-4d94-9c67-a633cdd11d57" />

## Etape 2 : Création de colonnes avec des formules excel
 J'ai ensuite réalisé la création des différentes colonnes en utilisant les formules suivantes :
- full_name  : =B2 & " DY3" & C2
- allow_to_force_purchase_qty  : = FAUX
- allow_to_force_sale_qty    : = FAUX
- stock_managed  : =SI(V2=1;VRAI;FAUX)
- dtype productTypeSelect  : =SI(V2=1;"Article";SI(V2=2;"Prestation";"Consignataire"))
- procurement_method_select  : = =RECHERCHEV(J2;[APPRO]APPRO!A:B;2;FAUX)
- product_category  : =RECHERCHEV(J2;[APPRO]APPRO!A:D;4;FAUX)
- product_family  : =RECHERCHEV(J2;[APPRO]APPRO!A:C;3;FAUX)
- purchase_currency  : =SI(OU(AG2="";SUPPRESPACE(SUBSTITUE(AG2;CAR(160);""))="");"EUR";AG2)

## Etape 3 : Générer le fichier XML
- J'ai tout d'abord créé le script python que voici :
--
import pandas as pd
from xml.etree.ElementTree import Element, SubElement, tostring
from xml.dom.minidom import parseString
df = pd.read_excel("Export.xlsx")
root = Element("data")
for _, row in df.iterrows():
    record = SubElement(root, "record", model="com.axelor.apps.base.db.Product")
    for col in df.columns:
        value = str(row[col]).strip()
        if value != "nan" and value != "":
            SubElement(record, "field", name=col).text = value
xml_str = parseString(tostring(root, encoding="utf-8")).toprettyxml(indent="  ")
with open("produits_axelor.xml", "w", encoding="utf-8") as f:
    f.write(xml_str)

print("✅ Fichier produits_axelor.xml généré avec succès !")
- Je l'ai ensuite exécuté dans l'invite de commandes puis le fichier a été géréné



