---
title: À propos
date:  "2021-01-05T20:00:00+09:00"
draft: false
---

# À propos de Asahi Linux

Asahi Linux est un projet et une communauté ayant pour objectif de porter Linux sur les Macs équipés de puces Apple Silicon, en commençant par les modèles 2020 du Mac Mini M1, MacBook Air et MacBook Pro.

Notre but n'est pas juste de faire fonctionner Linux sur ces machines mais de l'optimiser au point où il peut être utilisé comme système d'exploitation principal. Cela demande une énorme quantité de travail car les processeurs Apple Silicon n'ont aucune documentation. En particulier, nous réalisons une rétro-ingénierie de l'architecture GPU d'Apple en vue du développement d'un pilote open source pour celle-ci.

Asahi Linux est développé par une communauté dynamique de développeurs de logiciels open source passionnés.

## Le nom

Asahi signifie "le soleil levant" en Japonais, et c'est également le nom d'un cultivar de pommes. 旭りんご (*asahi ringo*) est ce que nous connaissons sous le nom des pommes McIntosh, la variété de pomme qui a donné son nom au Mac.

## Le logo

<img src="/img/AsahiLinux_logomark.svg" alt="Asahi Linux logo" width="100">

Le logo et le site web de Asahi Linux ont été conçus par [soundflora*](https://soundflora.tokyo). Vous pouvez trouver l'illustration du logo [ici](https://github.com/AsahiLinux/artwork/tree/main/logos).

# FAQ

## Quels appareils seront pris en charge ?

Tous les Macs avec Apple Silicon sont considérés, ainsi que les générations futures dans la mesure du temps de développement disponible. Nous prenons actuellement en charge la plupart des machines des générations M1 et M2.

## Est-ce une distribution Linux ?

Asahi Linux est un projet global de développement du support de Linux sur ces Macs. La majorité du travail réside dans le support matériel, les pilotes, et d'autres outils. Il sera ensuite intégré en amont aux projets concernés. Notre distribution phare actuelle est [Fedora Asahi Remix]({{< relLangURL "fedora" >}}), qui est une collaboration entre Asahi Linux et le projet Fedora, et sert à la fois de distribution optimisée pour l'utilisateur final et de référence pour d'autres distributions qui souhaitent intégrer notre travail.

D'autres distributions ont déjà commencé à travailler sur l'implémentation du support pour ces plateformes, et nous pensons qu'il y aura davantage d'options officiellement disponibles à l'avenir. Vous trouverez sur la page [Distributions Alternatives](https://github.com/AsahiLinux/docs/wiki/SW:Alternative-Distros) une liste des distributions intégrant actuellement notre projet.

## Apple autorise-t-il cela ? N'est-il pas nécessaire d'effectuer un jailbreak ?

Apple autorise le démarrage de noyaux non signés ou personnalisés sur les Macs équipés de la puce Apple Silicon sans nécessité de jailbreak ! Il ne s'agit pas d'un hack ou d'un oubli, mais d'une fonctionnalité réelle intégrée par Apple dans ces appareils. Cela signifie qu'à la différence des appareils iOS, Apple n'a pas l'intention de restreindre l'OS que vous pouvez utiliser sur les Macs (bien qu'ils ne faciliteront probablement pas le développement).

## Est-ce légal ?

Tant que aucun code n'est extrait de macOS pour construire le support Linux, le résultat est totalement légal à distribuer et à utiliser par les utilisateurs finaux, car il ne constituerait pas une œuvre dérivée de macOS. Veuillez consulter notre [Politique de droits d'auteur & Rétro-ingénierie]({{< relLangURL "copyright" >}}) pour plus d'informations.

## Comment cela sera-t-il publié ?

Tout le développement se déroule sur notre [GitHub](https://github.com/AsahiLinux). Toutes les contributions seront rédigées dans le but de les intégrer dans les projets respectifs (en commençant par le noyau Linux) et intégrées en amont dès qu'elles sont utilisables. Le code sera sous double licence, avec la licence en amont (par exemple, GPL) et une licence permissive (par exemple, MIT), afin de garantir que le travail puisse être réutilisé dans d'autres systèmes d'exploitation lorsque cela est possible.

## Cela rendra-t-il les Mac avec Apple Silicon une plateforme entièrement ouverte ?

Non, Apple contrôle toujours le processus de démarrage et, par exemple, le logiciel exécuté sur le Secure Enclave Processor. Cependant, aucun appareil moderne est "complètement ouvert" - aucun ordinateur utilisable avec logiciel et matériel complètement open source n'existe aujourd'hui (malgré les affirmations de certaines entreprises qui veulent se présenter ainsi). Ce qui est intéressant, c'est l'endroit où on trace la ligne entre les parties fermées et les parties ouvertes. La ligne sur les Macs équipés de puces Apple Silicon est lorsque l'image du noyau alternatif est démarrée, pendant que le logiciel du SEP reste fermé - ce qui est assez similaire avec la ligne des PCs standards, où le logiciel UEFI démarre le chargeur de l'OS, pendant que le logiciel du ME/PSP reste fermé. En fait, on pourrait soutenir que les plateformes x86 courantes sont potentiellement plus intrusives, car le micrologiciel UEFI propriétaire est autorisé à interrompre le processeur principal de l'OS à tout moment via des interruptions SMM, ce qui n'est pas le cas sur les Mac équipés de la puce Apple Silicon. Cela a des implications réelles en termes de performances/stabilité ; ce n'est pas simplement une question philosophique.

## Qui travaille sur Asahi Linux ?

Asahi Linux est une communauté, et tout le monde est invité à contribuer. Si vous êtes intéressé par la contribution, consultez notre [page de contribution]({{< relLangURL "contribute" >}}) ! Les contributeurs majeurs sont :

* [Hector Martin "marcan"](https://github.com/marcan), le chef de projet de Asahi. marcan est un rétro-ingénieur chevronné et développeur avec plus de 15 ans d'expérience dans le portage de Linux et l'exécution de logiciels non officiels sur des appareils non documentés et/ou fermés. C'est son projet le plus ambitieux à ce jour, et il finance l'effort grâce aux [dons de la communauté et au parrainage]({{< relLangURL "support" >}}). Ses précédents projets incluent [PS4 Linux](https://github.com/fail0verflow/ps4-linux), un port de Linux sur le matériel propriétaire de la PS4, capable d'accélération 3D complète utilisant OpenGL et Vulkan (pilotes radeon/amdgpu); [AsbestOS](https://github.com/marcan/asbestos), un bootloader PS3 Linux pour mode GameOS, associé à un patch de noyau pour faire fonctionner Linux sur la PS3 Slim; et de nombreuses contributions à l'[écosystème Wii Homebrew](https://wiibrew.org/), incluant faire partie de l'équipe qui a développé [The Homebrew Channel](https://wiibrew.org/wiki/Homebrew_Channel) et [BootMii](https://wiibrew.org/wiki/BootMii), en documentant la plupart du matériel, et en contribuant à mettre à disposition des outils de SDK pour homebrew.

* [Alyssa Rosenzweig](https://rosenzweig.io/), la responsable GPU de Asahi. Alyssa est une experte en piratage graphique sous Linux, reconnue pour son travail de rétro-ingénierie des GPU Arm Mali afin de construire le pilote libre Panfrost. Elle est une développeuse en amont pour Mesa3D, assurant la maintenance des pilotes Panfrost et Asahi Mesa.

* [Asahi Lina](https://github.com/asahilina), notre experte en source du noyau GPU. Lina a rejoint l'équipe pour effectuer une rétro-ingénierie de l'interface du noyau GPU M1 et s'est retrouvée à écrire le premier pilote de noyau GPU Linux en Rust au monde. Lorsqu'elle ne travaille pas sur le pilote de noyau Asahi DRM, elle travaille parfois sur des outils et une infrastructure open source pour les VTuber.

* [Dougall Johnson "dougallj"](https://github.com/dougallj), extraordinaire dans l'architecture des jeux d'instructions. Dougall a réalisé la plupart de la rétro-ingénierie du jeu d'instructions du GPU d'Apple et a analysé la synchronisation des cœurs CPU de l'Apple M1 pour déduire des détails micro-architecturaux.

* [Sven Peter](https://github.com/svenpeter42). Sven a travaillé sans relâche sur le support Linux en amont pour la Table de Résolution d'Adresse des Dispositifs (DART) d'Apple, nécessaire pour l'USB, le PCIe, l'Ethernet et le Wi-Fi. Il a également ajouté le support USB gadget à m1n1, et travaille actuellement sur le support de DisplayPort et Thunderbolt.

* [Mark Kettenis](https://github.com/kettenis), développeur OpenBSD. Mark a écrit m1n1 et des pilotes U-Boot pour les périphériques centraux de l'Apple M1, y compris la mise en service nécessaire pour PCIe et NVMe (ANS). Mark a également écrit des pilotes OpenBSD pour l'Apple M1 en tant qu'effort parallèle au portage Linux.

* [Martin Povišer](https://github.com/povik/), qui dirige nos efforts sur le pilote de noyau audio. Martin a écrit et est en cours de mise en amont des pilotes audio spécifiques au SoC d'Apple ainsi que des pilotes pour les codecs propriétaires d'Apple et leurs variantes.

* [Janne Grunau](https://github.com/jannau), qui a mis en œuvre le support du pavé tactile/clavier pour la série M1 et assure maintenant la maintenance du pilote du contrôleur d'affichage (DCP). Il a également été impliqué dans de nombreux autres aspects, notamment le nettoyage et la soumission de l'arbre des périphériques.
