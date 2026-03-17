# Challenge C502 17/03/2026

## 🧑‍🏫 Pitch de l’exercice

Le challenge du jour consiste à :

- Réaliser les [3 premiers lab de PortSwigger sur les LFI](https://portswigger.net/web-security/all-labs#path-traversal),
- Réaliser le [challenge LFI root-me](https://www.root-me.org/fr/Challenges/Web-Serveur/Local-File-Inclusion)
- Réaliser le [challenge LFI double encoding root-me](https://www.root-me.org/fr/Challenges/Web-Serveur/Local-File-Inclusion-Double-encoding).

[Cours C501.](/RESUME.md#-c501-introduction-au-pentesting--faille-xss)

> 📚 **Ressources** :
>
> - Portswigger explication du Cross-site scripting (XXS) : <https://portswigger.net/web-security/cross-site-scripting>
> - Owasp explication détaillée du path traversal : <https://owasp.org/www-community/attacks/Path_Traversal>
> - Référence complète sur toutes les techniques LFI/RFI  : <https://hacktricks.wiki/en/pentesting-web/file-inclusion/index.html>
> - Liste exhaustive de payloads classés par technique : <https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion>
> - Documentation officielle PHP sur tous les wrappers disponibles : <https://www.php.net/manual/fr/wrappers.php>

---

### 💉 Résolution : Lab PortSwigger - Server-Side Template Injection (SSTI)

<https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-basic>

**Contexte :**
Lors de la consultation du premier produit, un message "Rupture de stock" est généré sur la page d'accueil via une requête GET utilisant le paramètre `?message=`.

#### **Étape 1 : Preuve de concept (PoC)**

- **Analyse :** Le site utilise le moteur de template Ruby "ERB", dont la syntaxe pour exécuter du code est `<%= code %>`.
- **Test :** On vérifie si le serveur interprète nos instructions en lui soumettant un calcul mathématique simple : `<%= 7*7 %>`.
- **Injection :** On encode cette payload au format URL et on l'injecte : `?message=<%25%3d+7*7+%25>`
- **Résultat :** La page web affiche `49` à la place du message d'erreur. La vulnérabilité SSTI est confirmée : le serveur exécute aveuglément notre code.

#### **Étape 2 : Exploitation et RCE (Exécution de code à distance)**

- **Objectif du Lab :** Supprimer le fichier `morale.txt` situé dans le dossier de l'utilisateur Carlos.
- **Méthode :** Dans le langage Ruby, la fonction `system()` permet d'exécuter des commandes de système d'exploitation directement sur le serveur.
- **Payload brute :** `<%= system("rm /home/carlos/morale.txt") %>`
- **Payload finale (encodée URL) :** `<%25+system("rm+/home/carlos/morale.txt")+%25>`
- **Action :** En remplaçant la valeur du paramètre `message` par cette payload encodée, le serveur exécute la commande de suppression. Le lab est validé.

---
