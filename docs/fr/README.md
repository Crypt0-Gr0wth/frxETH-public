# Parcours francais : Frax Ether (frxETH)

Lecture commentee du depot FraxFinance/frxETH-public : le liquid staking token frxETH et son coffre de rendement sfrxETH.

Sommaire :

1. Presentation de Frax Ether
2. frxETH.sol et ERC20PermitPermissionedMint.sol
3. frxETHMinter : submit et le mint de base
4. depositEther et OperatorRegistry : la pile de validateurs
5. submitAndDeposit : mint et stake en une transaction
6. sfrxETH : le coffre ERC-4626 de staking
7. xERC4626 : la mecanique des cycles de recompenses
8. Gouvernance et pause : le modele a deux cles
9. Le withdrawal credential et le changement de validateurs
10. Comparaison avec les autres liquid staking tokens
11. Limites de securite documentees par l equipe
12. Le token DepositContract et les briques externes
13. Limites et perimetre de ce parcours

Ce parcours est une lecture pedagogique du code source, sans installation ni execution du projet.
