# Chapitre 5 — submitAndDeposit : mint et stake en une transaction

submitAndDeposit(recipient) combine les deux etapes (mint frxETH, puis staking pour sfrxETH) en un seul appel utilisateur. La sequence est : _submit(address(this)) mint le frxETH au contrat frxETHMinter lui-meme plutot qu a l utilisateur, puis frxETHToken.approve(sfrxETHToken, msg.value) autorise le coffre sfrxETH a depenser ce frxETH, puis sfrxETHToken.deposit(msg.value, recipient) execute le depot ERC-4626 standard et envoie les parts sfrxETH resultantes directement au destinataire final.

Un require(sfrxeth_recieved > 0) protege contre un depot qui reussirait techniquement mais ne produirait aucune part (cas degenere). Le commentaire du code note explicitement une limite connue : integrer EIP-712/EIP-2612 ici serait delicat a cause d une possible confusion entre msg.sender et tx.origin si ce contrat intermediaire etait remplace plus tard — un exemple de dette technique documentee plutot que cachee.

Du point de vue utilisateur, submitAndDeposit est le chemin le plus courant en pratique : la plupart des interfaces front-end de Frax orientent directement vers sfrxETH plutot que vers du frxETH brut sans rendement.
