# Documentation Technique - Module de Génération de Coupons

## 1. Vue d'ensemble
Cette page génère des rapports PDF de coupons selon deux formats (centrés ou à gauche) en fonction des critères suivants :
- Période (date début/fin)
- Émetteur(s) sélectionné(s)
- Format de coupon (stocké en session)

## 2. Structure du code

### Classe principale
```csharp
public partial class Coupons : System.Web.UI.Page
{
    public SqlConnection maConnexion = new SqlConnection();
    LoginUser userData = new LoginUser();
    
    protected void Page_Load(object sender, EventArgs e)
    {
        ReportDocument rdc = new ReportDocument();
        try {
            // Initialisation et traitement
        }
        catch (Exception Erreur) {
            // Gestion des erreurs
        }
        finally {
            maConnexion.Close();
        }
    }
}
```

## 3. Workflow principal

### 1. Initialisation
```csharp
maConnexion.ConnectionString = ConfigurationManager.AppSettings["ConnexionString"];
string dateDebut = Session["dateDebut"].ToString();
string dateFin = Session["dateFin"].ToString();
string emetteur = Session["emetteur"].ToString();
```

### 2. Sélection du modèle de rapport
```csharp
if (Session["coupons"].ToString() == "Centrés") {
    rdc.Load(Server.MapPath("../../report/Coupons_Centre.rpt"));
} else {
    rdc.Load(Server.MapPath("../../report/Coupons_Gauche.rpt"));
}
```

### 3. Exécution de la procédure stockée
```csharp
SqlCommand cmd = new SqlCommand("PS_Coupons", maConnexion);
cmd.Parameters.AddWithValue("@emetteur", emetteur);
cmd.Parameters.AddWithValue("@dateDebutDatetime", DateTime.Parse(dateDebut).ToString("yyyy-MM-ddTHH:mm:ss"));
cmd.Parameters.AddWithValue("@dateFinDatetime", DateTime.Parse(dateFin).ToString("yyyy-MM-ddTHH:mm:ss"));
cmd.CommandTimeout = 300; // 5 minutes timeout
```

### 4. Formatage des dates
```csharp
string finalDebut = $"{dateDebut.Substring(8, 2)}/{dateDebut.Substring(5, 2)}/{dateDebut.Substring(0, 4)} {dateDebut.Substring(11, 5)}";
// Format: JJ/MM/AAAA HH:mm
```

### 5. Génération du PDF
```csharp
rdc.SetDataSource(ds);
rdc.SetParameterValue("nom", Session["ID_UTIL"]);
rdc.PrintOptions.PaperOrientation = CrystalDecisions.Shared.PaperOrientation.Portrait;

byte[] byteArray = null;
using (Stream oStream = rdc.ExportToStream(ExportFormatType.PortableDocFormat)) {
    byteArray = new byte[oStream.Length];
    oStream.Read(byteArray, 0, Convert.ToInt32(oStream.Length - 1));
}

// Envoi au navigateur
Response.ContentType = "application/pdf";
Response.AddHeader("content-disposition", "inline; filename=Impayes_Definitifs.pdf");
Response.BinaryWrite(byteArray);
```

## 4. Procédure stockée

### `PS_Coupons`
**Paramètres** :
- `@emetteur` : Liste des codes émetteurs (format CSV)
- `@dateDebutDatetime` : Date de début (format ISO)
- `@dateFinDatetime` : Date de fin (format ISO)

**Sortie attendue** :
- Dataset compatible avec le rapport Crystal Reports

## 5. Gestion des erreurs

**Stratégie actuelle** :
```csharp
try {
    // Opérations principales
} 
catch (Exception Erreur) {
    string messageErreur = Erreur.ToString();
    // TODO: Journaliser l'erreur
}
finally {
    maConnexion.Close(); // Fermeture garantie
}
```

**Améliorations recommandées** :
```csharp
// Dans Web.config
<system.web>
    <customErrors mode="RemoteOnly" defaultRedirect="~/Error.aspx" />
</system.web>

// Journalisation
Logger.Error($"Erreur génération coupons: {Erreur.Message}");
```

## 6. Sécurité

**Mesures existantes** :
- Timeout explicite (300s)
- Fermeture garantie des connexions
- Gestion basique des erreurs

**Recommandations** :
```csharp
// Validation des entrées
if (!DateTime.TryParse(dateDebut, out _) || !DateTime.TryParse(dateFin, out _)) {
    throw new ArgumentException("Format de date invalide");
}

// Restriction d'accès
if (!userData.isAccess(2)) { // Niveau accès supérieur
    Response.Redirect("~/Unauthorized.aspx");
}
```

## 7. Performances

**Optimisations** :
- Utilisation de `CommandTimeout` pour les grosses requêtes
- Bufferisation de la réponse
- Libération des ressources via `using`

**Points d'attention** :
- Taille des PDF générés
- Charge serveur pour les gros exports

## 8. Exemple d'utilisation

1. **Page appelante** :
```csharp
Session["dateDebut"] = "2023-01-01";
Session["coupons"] = "Centrés"; // Ou "Gauche"
Response.Redirect("~/report/pages/Coupons.aspx");
```

2. **Sortie PDF** :
- Orientation portrait
- Nom de fichier : `Impayes_Definitifs.pdf`
- Affichage inline dans le navigateur

## 9. Évolution possible

**Nouvelles fonctionnalités** :
```csharp
// Ajout dans les paramètres
rdc.SetParameterValue("titrePersonnalise", 
    Session["TitrePersonnalise"] ?? "Rapport standard");

// Option d'export Excel
if (Session["exportExcel"] != null) {
    rdc.ExportToHttpResponse(
        ExportFormatType.Excel, 
        Response, 
        true, 
        "Coupons_" + DateTime.Now.ToString("yyyyMMdd"));
}
```

Cette documentation couvre l'ensemble des aspects techniques du module de génération de coupons, avec des pistes concrètes pour son amélioration et son optimisation.
