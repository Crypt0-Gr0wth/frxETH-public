# Frax Ether (frxETH) — Parcours francais

Depot source (upstream) : https://github.com/FraxFinance/frxETH-public

---

# Chapitre 1 — Presentation de Frax Ether

Frax Ether est le liquid staking token d ETH de l ecosysteme Frax. Contrairement a des protocoles comme Lido (stETH, rebasing) ou Rocket Pool (rETH, share-based), Frax Ether separe deliberement le token de depot du token de rendement en deux contrats distincts : frxETH et sfrxETH.

frxETH.sol est un token ERC-20 simple, non-rebasant, frappe 1 pour 1 contre de l ETH depose. Il ne genere aucun rendement par lui-meme : detenir du frxETH brut equivaut a detenir de l ETH sans le staker. Pour toucher le rendement du staking, l utilisateur doit deposer son frxETH dans le second contrat, sfrxETH.sol, un coffre ERC-4626 qui applique un taux de change croissant entre frxETH et sfrxETH au fil du temps.

Cette separation en deux tokens (mint token + vault token) plutot qu un unique token rebasant est le choix architectural central du protocole. Elle permet a frxETH de rester parfaitement compatible avec tout systeme DeFi qui ne gere pas bien les tokens rebasants (AMM, protocoles de pret), tout en offrant via sfrxETH un vehicule de rendement conforme au standard ERC-4626 largement supporte.

Le depot ne contient que 5 contrats non-triviaux (365 lignes de code au total selon le README du contest d audit) : ERC20PermitPermissionedMint.sol (parent de frxETH), frxETH.sol, frxETHMinter.sol, OperatorRegistry.sol et sfrxETH.sol. C est une base de code volontairement minimaliste, focalisee sur une seule fonction : convertir de l ETH en frxETH, puis optionnellement en sfrxETH.

Ce parcours s appuie sur le depot clone a la date d ecriture (FraxFinance/frxETH-public, branche master).

---

# Chapitre 2 — frxETH.sol et ERC20PermitPermissionedMint.sol

frxETH.sol lui-meme est un contrat presque vide : il herite entierement de ERC20PermitPermissionedMint et se contente de fixer le nom (Frax Ether) et le symbole (frxETH) dans son constructeur. Toute la logique vit dans le parent.

ERC20PermitPermissionedMint combine trois briques : ERC20Permit d OpenZeppelin (support EIP-2612, signatures hors-chaine pour approuver des depenses sans transaction separee), ERC20Burnable (permet de bruler des tokens), et Owned de Synthetix (modele de propriete simple avec un seul owner).

Le point cle est le systeme de mint permissionne : un tableau minters_array et un mapping minters trackent quelles adresses ont le droit d appeler minter_mint et minter_burn_from. Seul frxETHMinter sera ajoute comme minter autorise en pratique. addMinter et removeMinter sont proteges par le modifier onlyByOwnGov, qui autorise soit l owner soit une adresse timelock_address distincte — un modele de gouvernance a deux niveaux qu on retrouve identique dans OperatorRegistry.sol et frxETHMinter.sol.

removeMinter ne supprime pas vraiment l entree du tableau minters_array : elle la remplace par address(0) pour preserver les index des autres entrees, au prix de laisser des trous dans le tableau. C est un compromis gas/simplicite assez commun dans ce type de registre append-only.

---

# Chapitre 3 — frxETHMinter : submit et le mint de base

frxETHMinter.sol est le seul contrat autorise a frapper du frxETH. Son entree principale est submit(), une fonction payable qui accepte de l ETH et mint un montant equivalent de frxETH au sender via _submit().

_submit() est protegee par nonReentrant et par un flag submitPaused controlable par la gouvernance. Elle verifie que msg.value n est pas nul, puis appelle frxETHToken.minter_mint(recipient, msg.value) — un mint strictement 1 ETH pour 1 frxETH, sans aucune conversion de taux.

Le contrat expose trois variantes d entree : submit() (mint au sender), submitAndGive(recipient) (mint a une adresse tierce), et un receive() externe qui route tout ETH envoye directement au contrat vers _submit(msg.sender). Envoyer de l ETH nu au contrat frxETHMinter suffit donc a recevoir du frxETH.

Un mecanisme de retenue existe : withholdRatio (en precision 1e6 via RATIO_PRECISION) definit la fraction de chaque depot que le contrat garde en ETH plutot que de l envoyer vers le staking. currentWithheldETH accumule ce montant retenu, que la gouvernance peut deplacer via moveWithheldETH. Un withholdRatio de 0 (valeur par defaut au deploiement) signifie que 100% de l ETH depose est destine au staking.

---

# Chapitre 4 — depositEther et OperatorRegistry : la pile de validateurs

Une fois que suffisamment d ETH s est accumule dans frxETHMinter, quelqu un (generalement un bot, sans permission particuliere requise) appelle depositEther(max_deposits) pour effectivement staker cet ETH via le contrat officiel ETH 2.0 DepositContract.

Le calcul est simple : numDeposits = (balance du contrat - currentWithheldETH) / DEPOSIT_SIZE, avec DEPOSIT_SIZE fixe a 32 ether. Le parametre max_deposits permet de plafonner le nombre de depots traites en une seule transaction, pour eviter un depassement de gas si un tres gros montant s est accumule d un coup.

Chaque iteration de la boucle appelle getNextValidator(), une fonction interne d OperatorRegistry qui depile (pop) le dernier validateur du tableau validators — une structure LIFO (last-in, first-out), pas une file FIFO. Chaque Validator contient pubKey, signature et depositDataRoot ; le withdrawalCredential courant (curr_withdrawal_pubkey) est commun a tous les validateurs et stocke separement.

Le mapping activeValidators trackee par pubKey empeche de deposer une seconde fois 32 ETH sur un validateur deja actif, ce qui bloquerait ces fonds jusqu a l activation des retraits. OperatorRegistry expose aussi des fonctions de maintenance reservees a la gouvernance (addValidator, addValidators, removeValidator, swapValidator, popValidators, clearValidatorArray) — le commentaire du code precise explicitement que la validite des cles et signatures n est PAS verifiee on-chain pour economiser du gas ; c est une verification qui doit etre faite off-chain avant l ajout, le DepositContract officiel d Ethereum servant de derniere ligne de defense en cas d erreur.

---

# Chapitre 5 — submitAndDeposit : mint et stake en une transaction

submitAndDeposit(recipient) combine les deux etapes (mint frxETH, puis staking pour sfrxETH) en un seul appel utilisateur. La sequence est : _submit(address(this)) mint le frxETH au contrat frxETHMinter lui-meme plutot qu a l utilisateur, puis frxETHToken.approve(sfrxETHToken, msg.value) autorise le coffre sfrxETH a depenser ce frxETH, puis sfrxETHToken.deposit(msg.value, recipient) execute le depot ERC-4626 standard et envoie les parts sfrxETH resultantes directement au destinataire final.

Un require(sfrxeth_recieved > 0) protege contre un depot qui reussirait techniquement mais ne produirait aucune part (cas degenere). Le commentaire du code note explicitement une limite connue : integrer EIP-712/EIP-2612 ici serait delicat a cause d une possible confusion entre msg.sender et tx.origin si ce contrat intermediaire etait remplace plus tard — un exemple de dette technique documentee plutot que cachee.

Du point de vue utilisateur, submitAndDeposit est le chemin le plus courant en pratique : la plupart des interfaces front-end de Frax orientent directement vers sfrxETH plutot que vers du frxETH brut sans rendement.

---

# Chapitre 6 — sfrxETH : le coffre ERC-4626 de staking

sfrxETH.sol est le contrat qui transforme du frxETH statique en une position generatrice de rendement. Il herite de xERC4626 (une extension du standard ERC-4626 de Solmate), qui lui-meme herite de ERC4626.

Le constructeur fixe le nom (Staked Frax Ether), le symbole (sfrxETH), et un parametre cle : rewardsCycleLength, la duree en secondes d un cycle de distribution de recompenses (immutable, fixee au deploiement).

sfrxETH surcharge les quatre fonctions standards d un vault ERC-4626 — deposit, mint, withdraw, redeem — pour y ajouter le modifier andSync, qui declenche automatiquement syncRewards() si le cycle de recompenses en cours est termine (block.timestamp >= rewardsCycleEnd) avant d executer l operation demandee. Cela evite qu un utilisateur doive attendre qu un tiers appelle syncRewards() manuellement pour beneficier d un taux de change a jour.

pricePerShare() est un raccourci pratique qui retourne convertToAssets(1e18), c est-a-dire combien de frxETH vaut une unite de sfrxETH — la mesure directe du rendement accumule depuis le depot initial. depositWithSignature() ajoute un chemin de depot en une seule transaction via un permit EIP-2612 sur l asset (frxETH), evitant une transaction d approve separee.

---

# Chapitre 7 — xERC4626 : la mecanique des cycles de recompenses

La bibliotheque xERC4626 (empruntee au design de xERC20) est le coeur du calcul de rendement de sfrxETH. Son objectif est d eviter qu un depot de recompenses ponctuel ne fasse instantanement sauter le taux de change frxETH/sfrxETH, ce qui creerait une fenetre de MEV exploitable (deposer juste avant l injection de recompenses, retirer juste apres).

Le mecanisme repose sur quatre variables : storedTotalAssets (le solde comptabilise en dur), lastRewardAmount (le montant de la derniere injection de recompenses, en cours de deverrouillage), lastSync (l instant de la derniere synchronisation) et rewardsCycleEnd (la fin du cycle courant, toujours un multiple de rewardsCycleLength).

totalAssets() ne retourne pas simplement storedTotalAssets : si le cycle est termine, elle ajoute l integralite de lastRewardAmount ; sinon, elle ajoute une fraction lineaire calculee comme lastRewardAmount * (block.timestamp - lastSync) / (rewardsCycleEnd - lastSync). Le rendement apparait donc progressivement, seconde par seconde, plutot que d un coup.

syncRewards() est la fonction qui declenche un nouveau cycle : elle calcule nextRewards comme la difference entre le solde reel de frxETH detenu par le contrat et ce qui est deja comptabilise (storedTotalAssets + lastRewardAmount), autrement dit tout frxETH supplementaire envoye au contrat sfrxETH (par le multisig Frax, en pratique, correspondant au rendement de staking accumule) devient la nouvelle reserve a deverrouiller. Elle ne peut etre appelee qu apres la fin du cycle courant (sinon SyncError()), et calcule la fin du prochain cycle en arrondissant au multiple superieur de rewardsCycleLength — avec une garde qui rallonge le cycle d une periode complete si moins de 5% de sa duree restait avant l alignement naturel, pour eviter des cycles anormalement courts.

---

# Chapitre 8 — Gouvernance et pause : le modele a deux cles

Un motif se repete a l identique dans ERC20PermitPermissionedMint.sol, OperatorRegistry.sol et frxETHMinter.sol (via son heritage d OperatorRegistry) : le modifier onlyByOwnGov, qui autorise une action si msg.sender est soit owner (via Owned.sol de Synthetix) soit timelock_address, une adresse distincte destinee a heberger un contrat de timelock on-chain.

Ce modele a deux cles separe une gouvernance rapide (owner, souvent un multisig d equipe) d une gouvernance plus lente et transparente (timelock, qui impose un delai visible avant l execution d une action sensible). Les deux peuvent agir independamment sur les memes fonctions protegees, ce qui est un choix pragmatique plutot qu un vrai systeme de vote on-chain — Frax garde la gouvernance fine (parametres du protocole) hors du perimetre de ce depot, qui se concentre sur le mint/stake d ETH.

frxETHMinter expose deux flags de pause independants, submitPaused et depositEtherPaused, bascules via togglePauseSubmits et togglePauseDepositEther, tous deux proteges par onlyByOwnGov. Cette granularite permet par exemple de bloquer l acceptation de nouveaux depots ETH tout en continuant a permettre le staking de l ETH deja accumule, ou l inverse.

recoverEther et recoverERC20 sont des fonctions d urgence classiques pour recuperer des fonds envoyes par erreur au contrat, egalement proteges par onlyByOwnGov.

---

# Chapitre 9 — Le withdrawal credential et le changement de validateurs

curr_withdrawal_pubkey, stocke dans OperatorRegistry, est l adresse de retrait ETH 2.0 associee a tous les validateurs actuellement dans la pile. C est une valeur sensible : elle determine ou finiront les ETH retires (et les recompenses de consensus) de chaque validateur active par ce contrat.

setWithdrawalCredential() impose une contrainte stricte : le tableau validators doit etre vide (numValidators() == 0) avant de pouvoir changer cette valeur. La raison est cryptographique — chaque signature de validateur pre-signee (le champ signature dans la struct Validator) est generee off-chain pour un withdrawal credential precis ; en changer invaliderait silencieusement toutes les signatures deja stockees si des validateurs restaient dans la pile avec l ancienne valeur.

clearValidatorArray() est donc typiquement appelee juste avant un changement de withdrawal credential, ce qui supprime tous les validateurs en attente (ils devront etre re-ajoutes avec des signatures fraiches correspondant au nouveau credential).

swapValidator() et removeValidator(idx, dont_care_about_ordering) offrent deux strategies de suppression : un remove ordonne (couteux en gas, reconstruit tout le tableau) ou un swap-and-pop (efficace mais qui casse l ordre original) — un choix laisse a l appelant selon que l ordre des validateurs restants importe ou non.

---

# Chapitre 10 — Comparaison avec les autres liquid staking tokens

Le corpus de ce parcours couvre deja Lido (stETH) et Rocket Pool (rETH), ce qui permet de situer precisement le choix architectural de Frax.

Lido : stETH est un token rebasant — le solde de chaque detenteur augmente automatiquement a chaque rebase, ce qui pose des problemes d integration avec des protocoles DeFi qui ne gerent pas les changements de solde spontanes (d ou l existence de wstETH, un wrapper non-rebasant construit par-dessus).

Rocket Pool : rETH suit deja un modele a taux de change croissant (share-based), proche dans l esprit de sfrxETH, mais integre au sein d un seul token — pas de separation entre un token de depot et un token de rendement.

Frax : separe explicitement les deux roles des le depart. frxETH reste un ancrage de valeur simple et stable (1:1 avec l ETH deposant), pendant que sfrxETH, un coffre ERC-4626 distinct, porte seul la complexite du taux de change croissant. Cette separation permet a frxETH de servir de brique de base neutre (par exemple comme collateral dans un protocole de pret qui prefererait ne pas gerer un taux de change variable), tout en offrant sfrxETH comme vehicule de rendement optionnel et composable nativement avec l ecosysteme ERC-4626.

Le prix a payer est une etape supplementaire pour l utilisateur qui veut du rendement : deposer son frxETH dans sfrxETH, plutot que de detenir directement un token porteur de rendement des la conversion de l ETH.

---

# Chapitre 11 — Limites de securite documentees par l equipe

Le README du depot inclut une section Known Issues directement issue du contest d audit historique du protocole, ce qui donne une vision honnete des limites connues et deliberement acceptees.

Premier point : xERC4626 (dont sfrxETH herite) partage sa logique avec xTRIBE, qui presentait deux problemes de severite moyenne documentes publiquement (references code4rena M-01 et M-02) : certains utilisateurs peuvent ne pas pouvoir retirer avant la fin d un cycle de recompenses a cause d un underflow potentiel dans le calcul avant retrait, dans des cas limites.

Second point, deja mentionne au chapitre 4 : OperatorRegistry ne verifie aucune donnee de validateur on-chain (cle publique, signature) au moment de l ajout — cette verification est presumee faite hors-chaine par l equipe ou les operateurs autorises a appeler addValidator. Le DepositContract officiel d Ethereum 2.0 sert de garde-fou de dernier recours : il refusera un depot dont les donnees sont invalides, mais un validateur malveillant ou mal configure ajoute par erreur pourrait tout de meme bloquer une portion de fonds jusqu a resolution manuelle.

Le README precise egalement le perimetre exact de l audit historique : 5 contrats non-triviaux, 365 lignes de code au total, aucune AMM, aucun NFT, pas de fork d un projet existant, pas de logique de courbe mathematique nouvelle, deploiement mono-chaine uniquement.

---

# Chapitre 12 — Le token DepositContract et les briques externes

DepositContract.sol, present dans le depot, n est pas un contrat proprietaire de Frax : c est une copie du contrat officiel de depot ETH 2.0, deploye par la Fondation Ethereum, integre ici uniquement pour permettre de tester le flux complet localement (fork de mainnet dans les tests Foundry). Il n a pas ete audite dans le cadre de ce projet car il s agit d une infrastructure Ethereum core, deja largement verifiee independamment.

Owned.sol (cree par Synthetix.io) et une partie des contrats ERC20 heritent de bibliotheques externes deja largement auditees ailleurs — OpenZeppelin pour ERC20Permit et ERC20Burnable, Solmate pour le socle ERC4626 dont herite xERC4626. Le README classe explicitement ces briques dans les contrats inclus mais hors du perimetre d audit specifique au projet.

SigUtils.sol n est utilise que dans les tests (generation de signatures EIP-712 pour valider les flux de permit) et n a aucune presence en production.

Cette organisation — code proprietaire minimal, briques externes standard non reecrites — explique pourquoi la surface auditee de Frax Ether tient en seulement 365 lignes malgre un systeme complet de mint, staking et gestion de validateurs.

---

# Chapitre 13 — Limites et perimetre de ce parcours

Ce parcours couvre le coeur du systeme frxETH tel qu il existe dans le depot FraxFinance/frxETH-public a la date du clone : la creation du token frxETH, le processus de mint via frxETHMinter, la gestion de la pile de validateurs via OperatorRegistry, et le coffre de rendement sfrxETH avec sa mecanique de cycles xERC4626.

Sont volontairement laisses hors champ : toute la gouvernance plus large de l ecosysteme Frax (FXS, veFXS, le stablecoin FRAX lui-meme et ses mecanismes de collateralisation), qui vivent dans d autres depots et ne font pas partie de ce contest de code ; les scripts de deploiement (script/deployGoerli.s.sol, deployMainnet.s.sol) qui relevent de l operationnel plutot que de la logique du protocole ; les evolutions plus recentes de l ecosysteme Frax autour du staking (Frax v3, integrations cross-chain de frxETH) qui ne figurent pas dans cette version du depot clone ; et les details internes des bibliotheques externes (ERC4626 de Solmate, ERC20Permit d OpenZeppelin), traitees comme des briques deja documentees ailleurs.

L objectif reste le meme que pour les parcours precedents : donner une comprehension solide et verifiee du mecanisme central — ici, comment 32 ETH deposes se transforment en un validateur actif, et comment le rendement de ce validateur revient lineairement aux detenteurs de sfrxETH — sans pretendre couvrir l integralite de l ecosysteme Frax.

---

# Parcours francais : Frax Ether (frxETH)

Lecture commentee du depot FraxFinance/frxETH-public : le liquid staking token frxETH et son coffre de rendement sfrxETH.

Sommaire : Chapitre 1 Presentation de Frax Ether. Chapitre 2 frxETH.sol et ERC20PermitPermissionedMint.sol. Chapitre 3 frxETHMinter, submit et le mint de base. Chapitre 4 depositEther et OperatorRegistry, la pile de validateurs. Chapitre 5 submitAndDeposit, mint et stake en une transaction. Chapitre 6 sfrxETH, le coffre ERC-4626 de staking. Chapitre 7 xERC4626, la mecanique des cycles de recompenses. Chapitre 8 Gouvernance et pause, le modele a deux cles. Chapitre 9 le withdrawal credential et le changement de validateurs. Chapitre 10 comparaison avec les autres liquid staking tokens. Chapitre 11 limites de securite documentees par l equipe. Chapitre 12 le token DepositContract et les briques externes. Chapitre 13 limites et perimetre de ce parcours.

Ce parcours est une lecture pedagogique du code source, sans installation ni execution du projet.
