---
title: Droits d'auteur & Rétro-ingénierie
date:  "2021-01-05T20:00:00+09:00"
draft: false
---

# Politique de droits d'auteur 

Asahi Linux est un projet open source, et toutes les contributions doivent suivre les licences open source appropriées.

Ces règles de contribution sont particulièrement importantes pour le code qui doit être intégré en amont dans d'autres projets, afin de maintenir une traçabilité claire des licences.

## Licences

Le code développé spécifiquement pour Asahi Linux lui-même devrait être sous licence permissive, permettant à d'autres projets de bénéficier du code sans rencontrer de problèmes de compatibilité de licence. Les licences spécifiques seront décidées au cas par cas, mais nous utiliserons généralement une licence permissive comme la licence MIT.

Le code développé pour d'autres projets open source doit être sous licence selon la licence de ce même projet et doit suivre les conventions de licence/en-tête/auteur appropriées pour ce projet. Cependant, des modules spécifiques développés par les contributeurs d'Asahi Linux (comme des pilotes entiers ou des sous-modules) devraient être sous double licence, notamment sous une licence permissive comme MIT, afin de garantir qu'ils peuvent être portés ou réutilisés dans d'autres projets.

Pour le code original, nous utilisons des [identifiants de licence SPDX](https://spdx.github.io/spdx-spec/appendix-V-using-SPDX-short-identifiers-in-source-files/) pour enregistrer la licence des fichiers individuels de manière concise. Les fichiers devraient avoir un en-tête de la forme suivante (avec les informations de licence appropriées) :

```// SPDX-License-Identifier: GPL-2.0+ OR MIT```

Aucun auteur spécifique ne devrait être listé dans les fichiers source eux-mêmes, car cela est difficile à maintenir et peu probable de rester exact. Pour les emplacements de niveau supérieur et informatif où une déclaration de droits d'auteur est nécessaire, tels que le texte de la licence MIT ou les boîtes de dialogue de style "À propos", le code devrait être attribué à "The Asahi Linux Contributors".

Nous n'exigeons pas que les contributeurs acceptent un quelconque Accord de Licence de Contributeur (CLA), ni ne demandons une cession de droits d'auteur. Vous conservez la pleine propriété du droit d'auteur de tout code que vous écrivez. Ce ne sont que des conventions sur la manière dont l'origine du code devrait être documentée dans le contrôle de version et les fichiers.

## Attribution

Asahi Linux utilise Git pour la gestion du code source, et l'historique Git sert de registre des contributeurs. Le champ "Auteur" de Git doit refléter l'auteur principal d'une modification - si vous commitez une modification écrite par une autre personne, assurez-vous qu'elle soit listée comme l'auteur. Si une modification est écrite par plusieurs personnes, vous devriez ajouter une ou plusieurs lignes `Co-Developed-by: Foo Bar <foo@bar.com>` à la fin du message de commit.

Les versions du logiciel qui ne sont pas gérées par Git seront organisées de manière à avoir un fichier d'attribution généré automatiquement contenant la liste de tous les contributeurs Git.

Afin de certifier l'origine des contributions, tous les contributeurs doivent accepter le Certificat d'Origine du Développeur 1.1 (DCO) :

> ## Certificat d'Origine du Développeur 1.1 (DCO)
>
> En faisant une contribution à ce projet, je certifie que :
>
> * La contribution a été créée en totalité ou en partie par moi et que j'ai le droit de la soumettre sous la licence open source indiquée dans le fichier ; ou
> * La contribution est basée sur un travail précédent qui, d'après ma connaissance, est couvert par une licence open source appropriée et que j'ai le droit, en vertu de cette licence, de soumettre ce travail avec des modifications, que j'ai créées en totalité ou en partie, sous la même licence open source (sauf si j'ai la permission de soumettre sous une licence différente), comme indiqué dans le fichier ; ou
> * La contribution m'a été directement fournie par une autre personne qui a certifié (a), (b) ou (c) et que je ne l'ai pas modifiée.
> * Je comprends et accepte que ce projet et la contribution sont publics et qu'un enregistrement de la contribution (y compris toutes les informations personnelles que je soumets avec elle, y compris mon approbation) est conservé indéfiniment et peut être redistribué conformément à ce projet ou aux licences open source impliquées.

Pour certifier cela, ajoutez une ligne à la fin de votre message de commit Git comme suit :

```
Signed-off-by: Random J Developer <random@developer.example.org>
```

Cela peut être automatisé en utilisant simplement `git commit -s`.

Les noms réels ne sont pas nécessaires pour l'attribution ou les informations d'approbation. Nous encourageons les gens à utiliser un nom par lequel ils sont communément connus (par exemple, un nom que vous utilisez couramment pour interagir dans des espaces similaires), car cela contribue à établir la confiance.

# Politique de Rétro-ingénierie

Nous nous engageons à garantir que tout le code source et la documentation produits par le projet sont légaux et exempts de violations de droits d'auteur et d'autres problèmes juridiques. Pour garantir cela, nous avons une politique que tous les contributeurs doivent suivre, en particulier s'ils effectuent certains types de rétro-ingénierie.

L'ingénierie inverse en "salle blanche" est souvent considérée comme la norme de qualité pour garantir une situation juridique favorable à un projet de rétro-ingénierie. Cela implique d'avoir des équipes distinctes, l'une effectuant la rétro-ingénierie et rédigeant la documentation, et l'autre implémentant cette documentation dans le produit final. Cette approche n'est pas une exigence légale pour garantir que le produit final est exempt de violations de droits d'auteur, et elle ne garantit pas absolument un tel résultat, mais elle constitue une défense juridique assez solide en cas de questions relatives au droit d'auteur.

Nous reconnaissons qu'une approche en salle blanche véritable n'est pas viable pour la plupart des projets open source de cette nature. Ainsi, nous visons à garantir que le code et les contributions d'Asahi Linux sont effectivement équivalents à ce qu'une approche en salle blanche produirait, sans imposer les coûts d'un processus en salle blanche véritable.

Afin de protéger les contributeurs et les développeurs qui souhaitent éviter de tels sujets, nous exigeons que toutes les discussions sur la rétro-ingénierie aient lieu dans les canaux IRC #asahi-re et #asahi-gpu (pour l'ingénierie inverse du GPU).

## Développement non lié à la rétro-ingénierie

Si vous examinez simplement le code Asahi Linux existant et l'améliorez (sans prendre ni référencer de code ailleurs), vous n'effectuez pas de rétro-ingénierie, et vous n'avez pas à vous soucier d'autre chose. Assurez-vous simplement de suivre la [Politique de droits d'auteur](#politique-de-droits-dauteur).

## Référence à d'autres codes open source

Cela est généralement acceptable, mais vous ne devez pas copier-coller de code réel à moins que la licence soit compatible et que l'origine du code soit documentée.

En particulier, faites attention aux dumps open source d'Apple, car ils sont souvent sous licence APSL, qui n'est pas compatible avec GPL. Asahi Linux n'autorise pas l'utilisation directe de code sous licence APSL. Vous pouvez l'utiliser comme référence pour comprendre comment les choses fonctionnent, et vous pouvez vous inspirer des noms de registres (car ceux-ci sont essentiellement une documentation matérielle ; nous ne considérons pas les simples noms de registres de dernier niveau comme pouvant faire l'objet de droits d'auteur), mais ne copiez pas des blocs `#define` entiers ou des fichiers d'inclusion, ni d'autres codes, ni ne réimplémentez des flux de code ou des algorithmes identiques. Suivez des conventions sensibles pour les noms de registres en aval (telles que les préfixes).

Par exemple, [ce registre](https://github.com/opensource-apple/xnu/blob/master/pexpert/pexpert/arm64/arm64_common.h#L10), tel que documenté dans les fichiers d'inclusion de XNU, devrait être défini comme ceci lorsqu'il est utilisé pour Linux :

```
#define SYS_HID0                  sys_reg(3, 0, 15, 0, 0)
```

Ou encore :

```
#define SYS_APL_HID0              sys_reg(3, 0, 15, 0, 0)
```

Autrement dit, faire référence à du code sous une licence incompatible peut entraîner des problèmes similaires de droits d'auteur à ceux décrits ci-dessous. Veuillez vous assurer de lire cette section pour savoir ce qu'il ne faut pas faire.

Les outils internes destinés uniquement à l'exploration et qui ne seront pas distribués aux utilisateurs finaux sont autorisés à utiliser du code sous licence APSL, y compris à prendre directement des versions sous licence APSL et à les modifier, tant que toutes les exigences en matière de licence sont respectées. Par exemple, prendre une version sous licence APSL, la modifier pour aider à la rétro-ingénierie et utiliser le résultat dans macOS est acceptable, et nous pouvons héberger de tels changements dans un dépôt Asahi Linux, mais ils ne peuvent pas devenir une version réelle du projet.

**Soyez prudent avec le code open source qui fonctionne avec des formats de fichier, du matériel et des protocoles spécifiques d'Apple non documentés**. Par précaution, un tel code doit être traité comme s'il était sous une licence incompatible, similaire au code sous licence APSL. La raison en est que nous ne pouvons pas garantir que ce code n'est pas le produit d'une rétro-ingénierie qui violerait cette politique. Par conséquent, vous ne devez pas copier-coller ou incorporer directement un tel code, même si la licence est ostensiblement compatible. En règle générale, il est peu probable qu'incorporer directement un tel code offre plus qu'un gain de temps mineur ; ils sont plus utiles en tant qu'outils autonomes et en tant que code-documentation. Il est acceptable d'utiliser de tels outils autonomes tiers pendant le développement, ou de les fork et de les patcher pour l'exploration, comme le code sous licence APSL. Si un problème de droits d'auteur survient, nous pouvons facilement supprimer ces outils s'ils n'ont pas contaminé d'autres parties du code du projet. Les cas où il y a une valeur significative à tirer de l'intégration d'un tel code seront examinés et audités au cas par cas par les responsables du projet.

Autrement dit : nous voulons garantir non seulement la compatibilité des licences, mais aussi la compatibilité avec cette politique elle-même. Par conséquent, nous n'autorisons pas la combinaison de code développé en vertu de cette politique avec du code qui n'aurait pas été développé en vertu de cette politique dans les cas où une violation de la politique aurait pu se produire de manière plausible.

## Exploration matérielle

Une façon particulièrement sûre de faire de la rétro-ingénierie sans rencontrer de problèmes de droits d'auteur consiste simplement à sonder le matériel pour découvrir ce qu'il fait. Toute connaissance acquise de cette manière peut être utilisée en toute sécurité lors de l'écriture de code open source. Par exemple, le fait de lire des zones de registres, de manipuler des bits et de voir quels autres bits changent et comment le matériel se comporte. Cela est particulièrement utile pour compléter la connaissance d'une partie du matériel qui n'est que partiellement comprise, et peut souvent conduire à des connaissances qui ne sont pas utilisés et donc non présents dans le code binaire du fabricant.

## Traçage des registres et des données

Lorsque l'exploration à l'aveugle ne suffit pas, une autre astuce efficace consiste à exécuter macOS et à observer ce qu'il fait. Cela peut être réalisé en sauvegardant l'état des registres matériels avant et après une action, ou en interceptant les données entre différents composants, tels que la pile graphique de l'espace utilisateur et le pilote graphique du noyau, et en sauvegardant ou en modifiant les données.

Les valeurs des registres et les buffers de commandes ne sont généralement pas considérés comme des éléments protégeables par le droit d'auteur, il est donc sûr d'utiliser cette approche pour obtenir les informations nécessaires à l'écriture d'un pilote open source.

Les exceptions incluent les cas où du code ou de gros blocs de données sont téléchargés sur le matériel. Par exemple, un shader binaire obtenu en compilant votre propre code source de shader GPU ne serait normalement pas protégeable par le droit d'auteur. Cependant, s'il contient des sections significatives de code insérées par le compilateur (par exemple, des échafaudages importants ou des algorithmes pour mettre en œuvre une fonctionnalité particulière), alors cette partie peut l'être. Utilisez votre meilleur jugement dans ces cas, et ne copiez pas de longues séquences d'instructions qui ne correspondent pas directement au code que vous avez écrit sous forme source.

De même, si des données de logiciels sont téléchargés sur le matériel, ils ne peuvent pas être copiés tels quels, et nous devons déterminer comment les aborder au cas par cas.

## Désassemblage binaire et décompilation

Asahi Linux n'interdit pas le désassemblage binaire et la décompilation en tant que moyen d'obtenir les connaissances nécessaires à l'écriture d'implémentations open source. Cependant, c'est une voie dangereuse. Nous avons observé de nombreux cas de code "open source" développé par la simple traduction de code binaire obtenu par rétro-ingénierie. Cela est inacceptable et mettrait l'ensemble du projet ainsi que les projets amont en danger.

Étant donné que les responsables du projet Asahi Linux (actuellement marcan) sont ultimement responsables de certifier l'origine du code intégré en amont dans le cadre du projet, nous devons nous assurer que toute utilisation de cette approche suit un processus qui n'entraîne pas de violation du droit d'auteur. Nous avons vu précédemment d'autres projets accepter des contributions qui se sont révélées ne pas être propres, plaçant ces projets dans un doute juridique - cela était souvent découvert longtemps après la contribution du code, car l'origine du code n'est pas toujours immédiatement apparente. Pour cette raison, nous n'acceptons actuellement pas les contributions de code développé simultanément avec la rétro-ingénierie binaire, sauf de la part de contributeurs spécifiques en qui nous avons confiance. Veuillez contacter marcan pour plus de détails.

Tous les autres contributeurs sont tenus de suivre l'approche classique de la "clean-room" : si vous souhaitez désassembler ou décompiler des composants afin de contribuer au projet, vous devez prendre cette décision pour un composant/secteur spécifique avec précaution. Une fois que vous l'avez fait, on s'attend à ce que vous contribuiez uniquement à cette zone en rédigeant une documentation claire du matériel, laissant ainsi aux autres membres du projet le soin de l'implémenter.

Une précaution particulière doit être prise pour la partie graphique : les contributeurs **ne doivent pas désassembler ni décompiler les binaires des pilotes graphiques en espace utilisateur**, y compris le compilateur de shaders basé sur LLVM et Metal lui-même, même dans le but de produire une documentation pour le développement propre de Mesa. La raison en est double. Premièrement, la traçabilité des données en boîte noire est une technique très efficace qui peut être utilisée pour la rétro-ingénierie de manière isolée, rendant le désassemblage inutilement risqué. Deuxièmement, le code en espace utilisateur tend à être plus algorithmique et original par nature, comparé au code du noyau, rendant ces techniques particulièrement risquées. N'oubliez pas, l'objectif est d'effectuer une rétro-ingénierie du matériel GPU, et non du logiciel du pilote propriétaire.

Les contributeurs qui effectuent de la rétro-ingénierie binaire sont responsables de toutes les conséquences légales de leur travail, y compris les conséquences de la licence associée audit code.

**Important** : Afin d'assurer la sécurité juridique des membres du projet qui ne souhaitent pas s'impliquer dans cette approche de rétro-ingénierie, *toutes* les discussions à ce sujet sont autorisées **uniquement** dans le salon IRC #asahi-re. Cela sera strictement appliqué, et tout travail impliquant une rétro-ingénierie binaire sur d'autres salons #asahi-* entraînera une exclusion ou un banissement. De plus, ne collez aucun code décomplié/désassemblé dans les canaux IRC. Ces salons sont enregistrés publiquement.

## Utilisation de matériaux non publiés

Asahi Linux interdit absolument l'utilisation de tout matériel protégé par le droit d'auteur qui n'est pas disponible pour le public pendant la rétro-ingénierie. Cela inclut tout logiciel divulgué (sous forme source ou binaire), toute documentation non publiée, les versions non publiques (telles que les versions bêta restreintes), etc. Les contributeurs du projet sont tenus de s'abstenir d'acquérir ou d'utiliser un tel contenu. Seuls les matériaux explicitement rendus disponibles au grand public peuvent être utilisés. Cela s'applique aux matériaux provenant à la fois d'Apple et de tout autre tiers.
