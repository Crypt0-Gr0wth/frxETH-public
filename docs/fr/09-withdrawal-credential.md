# Chapitre 9 — Le withdrawal credential et le changement de validateurs

curr_withdrawal_pubkey, stocke dans OperatorRegistry, est l adresse de retrait ETH 2.0 associee a tous les validateurs actuellement dans la pile. C est une valeur sensible : elle determine ou finiront les ETH retires (et les recompenses de consensus) de chaque validateur active par ce contrat.

setWithdrawalCredential() impose une contrainte stricte : le tableau validators doit etre vide (numValidators() == 0) avant de pouvoir changer cette valeur. La raison est cryptographique — chaque signature de validateur pre-signee (le champ signature dans la struct Validator) est generee off-chain pour un withdrawal credential precis ; en changer invaliderait silencieusement toutes les signatures deja stockees si des validateurs restaient dans la pile avec l ancienne valeur.

clearValidatorArray() est donc typiquement appelee juste avant un changement de withdrawal credential, ce qui supprime tous les validateurs en attente (ils devront etre re-ajoutes avec des signatures fraiches correspondant au nouveau credential).

swapValidator() et removeValidator(idx, dont_care_about_ordering) offrent deux strategies de suppression : un remove ordonne (couteux en gas, reconstruit tout le tableau) ou un swap-and-pop (efficace mais qui casse l ordre original) — un choix laisse a l appelant selon que l ordre des validateurs restants importe ou non.
