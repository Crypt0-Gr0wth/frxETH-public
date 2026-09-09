# Chapitre 2 — frxETH.sol et ERC20PermitPermissionedMint.sol

frxETH.sol lui-meme est un contrat presque vide : il herite entierement de ERC20PermitPermissionedMint et se contente de fixer le nom (Frax Ether) et le symbole (frxETH) dans son constructeur. Toute la logique vit dans le parent.

ERC20PermitPermissionedMint combine trois briques : ERC20Permit d OpenZeppelin (support EIP-2612, signatures hors-chaine pour approuver des depenses sans transaction separee), ERC20Burnable (permet de bruler des tokens), et Owned de Synthetix (modele de propriete simple avec un seul owner).

Le point cle est le systeme de mint permissionne : un tableau minters_array et un mapping minters trackent quelles adresses ont le droit d appeler minter_mint et minter_burn_from. Seul frxETHMinter sera ajoute comme minter autorise en pratique. addMinter et removeMinter sont proteges par le modifier onlyByOwnGov, qui autorise soit l owner soit une adresse timelock_address distincte — un modele de gouvernance a deux niveaux qu on retrouve identique dans OperatorRegistry.sol et frxETHMinter.sol.

removeMinter ne supprime pas vraiment l entree du tableau minters_array : elle la remplace par address(0) pour preserver les index des autres entrees, au prix de laisser des trous dans le tableau. C est un compromis gas/simplicite assez commun dans ce type de registre append-only.
