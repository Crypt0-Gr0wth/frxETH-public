# Chapitre 3 — frxETHMinter : submit et le mint de base

frxETHMinter.sol est le seul contrat autorise a frapper du frxETH. Son entree principale est submit(), une fonction payable qui accepte de l ETH et mint un montant equivalent de frxETH au sender via _submit().

_submit() est protegee par nonReentrant et par un flag submitPaused controlable par la gouvernance. Elle verifie que msg.value n est pas nul, puis appelle frxETHToken.minter_mint(recipient, msg.value) — un mint strictement 1 ETH pour 1 frxETH, sans aucune conversion de taux.

Le contrat expose trois variantes d entree : submit() (mint au sender), submitAndGive(recipient) (mint a une adresse tierce), et un receive() externe qui route tout ETH envoye directement au contrat vers _submit(msg.sender). Envoyer de l ETH nu au contrat frxETHMinter suffit donc a recevoir du frxETH.

Un mecanisme de retenue existe : withholdRatio (en precision 1e6 via RATIO_PRECISION) definit la fraction de chaque depot que le contrat garde en ETH plutot que de l envoyer vers le staking. currentWithheldETH accumule ce montant retenu, que la gouvernance peut deplacer via moveWithheldETH. Un withholdRatio de 0 (valeur par defaut au deploiement) signifie que 100% de l ETH depose est destine au staking.
