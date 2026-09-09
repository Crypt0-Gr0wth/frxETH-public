# Chapitre 11 — Limites de securite documentees par l equipe

Le README du depot inclut une section Known Issues directement issue du contest d audit historique du protocole, ce qui donne une vision honnete des limites connues et deliberement acceptees.

Premier point : xERC4626 (dont sfrxETH herite) partage sa logique avec xTRIBE, qui presentait deux problemes de severite moyenne documentes publiquement (references code4rena M-01 et M-02) : certains utilisateurs peuvent ne pas pouvoir retirer avant la fin d un cycle de recompenses a cause d un underflow potentiel dans le calcul avant retrait, dans des cas limites.

Second point, deja mentionne au chapitre 4 : OperatorRegistry ne verifie aucune donnee de validateur on-chain (cle publique, signature) au moment de l ajout — cette verification est presumee faite hors-chaine par l equipe ou les operateurs autorises a appeler addValidator. Le DepositContract officiel d Ethereum 2.0 sert de garde-fou de dernier recours : il refusera un depot dont les donnees sont invalides, mais un validateur malveillant ou mal configure ajoute par erreur pourrait tout de meme bloquer une portion de fonds jusqu a resolution manuelle.

Le README precise egalement le perimetre exact de l audit historique : 5 contrats non-triviaux, 365 lignes de code au total, aucune AMM, aucun NFT, pas de fork d un projet existant, pas de logique de courbe mathematique nouvelle, deploiement mono-chaine uniquement.
