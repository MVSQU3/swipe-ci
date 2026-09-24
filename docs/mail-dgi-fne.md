# Brouillon — mail à support.fne@dgi.gouv.ci

**À :** support.fne@dgi.gouv.ci
**Objet :** Interfaçage API FNE — éditeur de logiciel de facturation pour micro-entreprises

---

Madame, Monsieur,

[Raison sociale], société basée à Abidjan, développe **Swipe CI**, une application mobile de facturation destinée aux micro-entreprises (régime RMI). L'application émettra des factures normalisées électroniques (FNE) via l'API décrite dans votre procédure d'interfaçage de mai 2025.

Avant de finaliser notre développement, nous souhaiterions obtenir vos précisions sur les points suivants :

1. **Validation du logiciel** : la validation de conformité (envoi de spécimens) porte-t-elle sur chaque entreprise utilisatrice, ou un éditeur peut-il faire valider son logiciel une seule fois, de sorte que ses clients n'aient ensuite qu'à activer leur clé API ?
2. **Éligibilité des micro-entreprises** : une entreprise au régime RME/RMI peut-elle opter pour le mode API en utilisant un logiciel édité par un tiers ?
3. **Consultation d'une facture** : existe-t-il un endpoint permettant de vérifier si une facture a bien été certifiée (par exemple après une coupure réseau pendant l'appel `/external/invoices/sign`) ? À défaut, quelle procédure recommandez-vous pour éviter une double certification ?
4. **Client particulier anonyme** : les champs `clientPhone` et `clientEmail` étant obligatoires, quelles valeurs faut-il renseigner lorsque le client (B2C) ne communique ni téléphone ni e-mail ?
5. **Remise** : le champ `discount` (facture et article) est-il exprimé en pourcentage ou en montant ?
6. **Stickers** : comment les stickers sont-ils obtenus et rechargés par l'entreprise, et à quel coût ?

Par ailleurs, nous souhaitons **inscrire [raison sociale] sur l'environnement de test** afin de démarrer les essais de facture de vente et d'avoir. Pourriez-vous nous indiquer la marche à suivre ?

Nous vous remercions par avance pour votre retour et restons à votre disposition pour toute information complémentaire.

Veuillez agréer, Madame, Monsieur, l'expression de nos salutations distinguées.

[Nom, fonction]
[Raison sociale] — [NCC]
[Téléphone] — [E-mail]
