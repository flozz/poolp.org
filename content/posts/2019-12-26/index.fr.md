---
title: "SPF-aware greylisting et filter-greylist"
date: 2019-12-26 14:15:00 +0200
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
    - le greylisting est une bonne idée
    - ce n'est pas très pratique aujourd'hui
    - beaucoup de gens se passent du greylisting ou trouvent des contournements
    - le SPF-aware greylisting rend le greylisting utilisable à nouveau
{{< /tldr >}}

# Shout out a mes sponsors &#x2764;&#xfe0f;

Un **GRAND MERCI** à mes sponsors sur [github](https://github.com/sponsors/poolpOrg)
et [patreon](https://www.patreon.com/gilles):
votre soutien est grandement apprécié !

Où est-ce que j'ai déjà lu ça ?
--
Cet article est une traduction de l'article "[SPF-aware greylisting and filter-greylist](/posts/2019-12-01/spf-aware-greylisting-and-filter-greylist/)",
publié sur ce blog début Décembre.

C'est mon troisième article rédigé en français depuis des années, je vous demande donc un peu d'indulgence :
si vous trouvez des fautes,
vous pouvez me les remonter pour que je les corrige,
ou faire une [pull request](https://github.com/poolpOrg/poolp.org/) pour les techies.

N'hésitez pas à **partager la publication** à l'aide des icônes en fin d'article et à la commenter <3

Bonne lecture !


Les erreurs SMTP en quelques mots
--
SMTP est un protocole **fail-safe** qui essaie de son mieux de garantir que les messages ne se perdent pas en transit.
Parmi les différents mécanismes et exigences du standard on trouve l'utilisation d'_échecs temporaires_.

Concrètement,
un nœud SMTP a **deux** états finaux pour un message :
soit le message **est distribué** et le nœud destination le détient,
soit il **n'est PAS distribué** et le nœud destination ne le détient pas.
Dans le second cas,
le protocole SMTP exige que le dernier nœud en charge notifie l'émetteur qu'une erreur est survenue.
Soit en rejetant le message lors de la session,
soit à l'aide d'une mail d'erreur différé (aussi connu sous le nom de `MAILER-DAEMON`).

Enfin,
il y a un **troisième** état qui n'est pas final : l'`échec temporaire`.
Il couvre tous les cas d'erreurs qui ont pu **temporairement** empêcher la distribution…
mais qui **pourraient** tout aussi bien ne plus se produire si **quelque chose** était corrigé sur le nœud de destination.

Contrairement aux mails distribués,
que les destinataires voient,
et aux mails échoués,
que les émetteurs voient,
les mails en `échec temporaire` ne sont _généralement_ pas vus par les utilisateurs.
Ils sont traités entre les _Mail eXchangers_ (MX) qui tentent de se les retransmettre en arrière-plan.
Il ne sont _généralement_ vus par les utilisateurs que lorsque les tentatives finissent par réussir ou qu'elles échouent définitivement après que leur temps de vie a expiré.

Dans le premier cas,
le destinataire va recevoir le message un peu plus tard que prévu,
le plus souvent sans même avoir su que des tentatives de retransmission ont eu lieu.
Dans le second cas,
l'émetteur recevra une notification lui indiquant que,
malgré les retransmissions,
le message n'a pas pu être distribué dans le délai imparti.

Les stratégies de retransmission en cas d'échec temporaire sont gérées différemment dans les différents logiciels,
mais vous pouvez postuler qu'il y aura une retransmission dans les secondes ou minutes qui suivent un échec temporaire.
Une approche populaire,
l'**incrément quadratique du délai**,
crée un délai initialement court mais qui augmente avec le nombre de tentatives en échec de sorte que plus un hôte est injoignable longtemps,
plus les tentatives vers cet hôte sont espacées.

Les échecs temporaires arrivent **tout le temps**,
c'est quelque chose de normal dans le protocole SMTP,
et en regardant les logs on verra très souvent des lignes telles que :

    4b3a6c195c1f6010 mta delivery evpid=8f26ea98359eccea from=<misc+bounces-4541-[redacted]=tin.it@opensmtpd.org> to=<[redacted]@tin.it> rcpt=<-> source="45.76.46.201" relay="62.211.72.32 (smtp.tin.it)" delay=0s result="TempFail" stat="421 <[redacted]@tin.it> Service not available - too busy"

Il ne s'agit pas d'erreurs que l'on ne trouve que chez des petits serveurs personnels.
Je connais deux Big Mailer Corps qui ont des machines en échec temporaire à peu près en permanence,
ils ont juste suffisamment de MX pour qu'une retransmission ait peu de chances de taper le même MX deux fois de suite.
Les mails sont généralement distribués à la première ou seconde tentative.

**Petit secret rien qu'entre nous** :
certains Big Mailer Corps se servent **volontairement** de ces échecs temporaires pour **évaluer la réputation des émetteurs**.
Ils génèrent ces erreurs à des moments clefs qui leurs permettent certaines analyses sur lesquelles je reviendrai dans un futur article.

Il y a beaucoup de choses à dire sur les erreurs SMTP,
mais résumons l'essentiel pour cet article :

Il existe un mécanisme dans le protocole SMTP qui permet à un MX destinataire de demander à un MX émetteur de retransmettre un message.
La retransmission a lieu en arrière-plan,
sans aucune action des utilisateurs,
et elle peut provoquer un délai dans la distribution.


Le greylisting expliqué
--
Le spam est une industrie du **volume**.

Les spammeurs ne sont pas intéressés par cibler des destinataires **spécifiques**.
Ils sont intéressés par cibler un **grand nombre** de destinataires pour que **statistiquement** cela augmente le nombre de destinataires qui se feront avoir.

Je ne vais pas trop creuser ce sujet parce que c'est l'objet d'un autre article en rédaction,
néanmoins vous devez avoir à l'esprit qu'il y a différents types de spammeurs.
Alors que certains utilisent des MX tout ce qu'il y a de plus normaux qui honorent les demandes de retransmissions,
beaucoup d'autres utilisent des outils ou des MX customisés pour **NE PAS** retransmettre.

**Honorer les retransmissions ralentit considérablement l'émission pour un résultat incertain**.
Garder trace de tous les destinataires en échec temporaire pendant des heures ou des jours pour pouvoir les retenter est loin d'être intéressant,
surtout s'il n'y a pas de certitude que la retransmission n'aboutira pas en un échec permanent de type "cet utilisateur n'existe pas".
Il faut conserver un état pour se rappeller des tentatives en échec et conserver les destinataires dans la queue d'envoi.
Directement ou indirectement cela impacte la capacité à envoyer en masse.

L'idée derrière le greylisting est très simple :

Le greylisting déclenche un échec temporaire pour les MX auquels vous ne faites pas encore confiance et garde trace des tentatives de retransmission.
Si la retransmission a lieu trop tôt elle est ignorée,
les spammers ne peuvent pas faire une retransmission immédiate et **doivent** garder un état.
Si la retransmission a lieu trop tard…
et bien c'est trop tard,
l'implémentation peut faire un bon nombre de choses depuis l'ajout en blacklist à la dégradation de la réputation,
en passant par un nouveau tour de greylisting, encore et encore et encore, …
Le résultat est que pour passer le greylisting,
les spammeurs sont **contraints de conserver un état** et de se comporter comme des implémentations correctes du point de vue de la RFC.
C'est en conflit direct avec l'économie de l'envoi de masse.

D'un autre côté,
les MX qui respectent la RFC vont naturellement retenter de nombreuses fois et finir par faire une tentative dans la bonne fenêtre de temps pour être whitelistés.


Le greylisting ne permet pas de se prémunir des spammeurs qui ont compromis des MX légitimes,
ni de ceux qui envoient à l'aide d'outils qui acceptent le coût de la retransmission.
Il permet par contre d'éliminer la plus grosse partie du spam :
celle qui provient de machines infectées et transformées en bots à spam,
ou de scripts dont le seul but est d'envoyer un maximum possible avant qu'un hébergeur ou un administrateur ne s'en rende compte.



Comment le greylisting fonctionne habituellement
--
Il y a différentes solutions de greylisting et elles ne fonctionnent pas toutes pareil.

L'idée générale est que vous avez une base de donnée qui garde trace des demande de retransmissions et de la fenêtre de temps durant laquelle une tentative est considérée comme valide pour un MX.
En revanche,
la manière dont sont comptabilisées les tentatives d'un MX varie d'une solution à une autre.
Certaines solutions ne regardent que l'adresse IP source,
alors que d'autres vont également regarder le MAIL FROM et/ou le RCPT TO.
Dans certains cas que j'ai pu observer,
la solution regardait également l'entête Message-ID.

La solution `spamd(8)` d'OpenBSD va par exemple garder trace de l'adresse IP source,
le nom d'hôte utilisé lors de la phase d'identification HELO/EHLO,
l'émetteur d'envelope (MAIL FROM) et le destinataire d'envelope (RCPT TO).
Pour passer le greylisting,
un MX doit faire une retransmission dans la bonne fenêtre de temps,
en provenance de la même IP source,
en utilisant le même nom d'hôte à l'identification,
depuis le même émetteur et pour le même destinataire.

Le problème avec le greylisting et les Big Mailer Corps
--
Pour de petits émetteurs,
le greylisting marche très bien parce que la plupart utilisent le même MX pour le trafic entrant et sortant.
Le MX vous contacte,
il lui est demandé de retenter plus tard,
il est accepté lorsqu'il revient une minute plus tard.

Ensuite,
vous avez Gmail / Yahoo / Outlook / …
qui non seulement disposent de MX différents pour le trafic entrant et sortant,
mais ont également des douzaines et des douzaines de MX sortants.
La situation devient la suivante :
un MX vous contacte,
il lui est demandé de retenter plus tard,
mais vous ne voyez jamais de retransmission de ce MX puisque c'est un autre qui a retenté et à qui il a été demandé de retenter plus tard,
etc…

Le greylisting avec ces hôtes est inutilisable et résulte en des mails qui peuvent arriver avec plusieurs heures ou jours de retard,
s'ils n'expirent pas tout simplement avant de vous atteindre.

Mais en même temps,
le greylisting empêche une grande quantité de nuisibles de vous atteindre en tuant le trafic de presque tous les scripts et les ordinateurs compromis.
Donc pour le conserver,
un grand nombre d'opérateurs de mails mettent en place des bricolages pour pouvoir continuer à utiliser le greylisting pour les petits émetteurs,
tout en ne l'appliquant pas pour les Big Mailer Corps.


Stratégies de contournement
--
J'utilise le pluriel mais en réalité toutes les stratégies reposent sur la même idée :
le **whitelisting**.

Le contournement consiste à trouver la liste des adresses IP des Big Mailer Corps pour leur donner un passe droit et leur permettre d'éviter le greylisting.

Est-ce que cela met en défaut le greylisting ?
Pas vraiment.
Le greylisting a pour but d'empêcher les MX ne respectant pas la RFC de vous atteindre,
mais les Big Mailer Corps utilisent des MX qui respectent la RFC.
Lorsque la retransmission a bien lieu plusieurs fois par le même MX sortant,
ils passent le test du greylisting sans souci.

Très bien,
ajoutons-les à la whitelist alors…
mais comment ?

Pendant un moment,
j'ai utilisé une approche naïve consistant à whitelister le `/24` de n'importe quel MX de Big Mailer Corps qui se retrouvait greylisté.
Après quelques temps,
ma liste était suffisamment grande pour que de nouveaux MX soient rarement hors de ma whitelist.
C'était pénible,
ça ne couvrait pas le cas de nouveaux MX me contactant depuis de nouvelles plages d'adresses IP,
et j'ai fini par aggréger les listes de différentes personnes à la mienne pour être sûr d'en avoir le plus possible.
Le résultat était une **whitelist en augmentation constante**.

Ce n'était pas très malin parce que la plupart des Big Mailer Corps utilisent SPF.
Ils ont un enregistrement DNS qui liste avec quelles adresses IP ils nous contactent,
de façons à ce que lorsque l'on reçoit un mail avec une adresse e-mail en provenance de chez eux,
on puisse vérifier que c'est bien un de leurs MX autorisé qui l'a transmis.
Avec ça en tête,
j'ai écrit `spfwalk` (maintenant une sous-commande de `smtpctl`) en Janvier 2018,
un outil qui permet de faire une énumération dans l'enregistrement SPF d'un domaine et extraire un maximum d'adresse IP possible.
Il y a un [article en anglais à propos de spfwalk](/posts/2018-01-08/spfwalk/) sur ce blog si ça vous intéresse.
L'outil avait pour vocation de remplacer mon approche naïve,
il est loin d'être parfait,
j'ai fait _mon mieux_.

La façon dont fonctionne SPF permet de vérifier facilement si une adresse IP est autorisée,
mais ne permet pas d'extraire facilement une liste d'adresse pour une vérification ultérieure.
Par exemple,
certaines politiques SPF s'attendent à une résolution DNS inverse de l'adresse IP source pour voir si le nom d'hôte corresponds au domaine d'envoi.
Dans ce type de politique SPF,
il n'y a _aucune_ adresse IP à extraire pour mettre dans une whitelist.
Donc…
l'énumération SPF fonctionne à peu près tant que l'on ne tombe pas sur les politiques qui ne peuvent pas être énumérées.
Pas de bol pour nous,
l'un des Big Mailer Corps est dans le périmètre.

Un grand nombre de personnes sont très contentes avec l'énumération SPF.
Elle améliore considérablement la situation,
mais elle reste un bricolage à mon sens et je n'utilise plus cet outil parce qu'il ne me semble pas être la bonne approche.

La preuve de concept `filter-greylist`
--
Le mois dernier,
j'ai écrit comme preuve de concept pour OpenSMTPD [un filtre qui utilise une approche différente](/posts/2019-11-17/november-2019-report-opensmtpd-6.6.1p1-filter-greylist-and-tons-of-portable-cleanup/)
et qui est disponible
[sur Github](https://github.com/poolpOrg/filter-greylist).

Je vais juste clarifier avant tout que **je ne prétends pas avoir inventé cette méthode**,
elle m'a juste semblée être une bonne approche alors je l'ai implémentée.
Il se peut qu'on trouve d'autres implémentations de cette même idée ailleurs,
j'avoue que **je n'ai pas vraiment cherché très fort** (pour ne pas dire "du tout").
Je ne sais donc pas s'il existe d'autres implémentations similaires,
et si elles existent je ne sais pas si mon implémentation exploite l'idée de la même manière.

L'idée de base est que nous ne voulons pas vraiment que les Big Mailer Corps soient exemptés de greylisting,
ils en sont exemptés parce que l'on ne sait pas considérer tous leurs MX sortants comme identiques dans un test de greylisting.
En faisant une énumération SPF pour extraire leurs adresses IP,
notre intention est simplement de considérer que toutes les adresses IP de tous leurs MX sortants sont identiques pour nous :
elles devraient pouvoir être interchangeables dans un test de greylisting.
En d'autres termes,
ce que l'on veut c'est un **SPF-aware greylisting** (greylisting conscient de SPF).

La preuve de concept considère qu'il y a deux types d'émetteurs :
ceux qui publient des enregistrements SPF et ceux qui n'en publient pas.

Les MX qui **ne publient pas d'enregistrement SPF** sont **greylistés par adresse IP source**,
comme cela serait le cas avec un greylisting traditionnel.
Commes les Big Mailer Corps requièrent que les émetteurs fournissent un enregistrement SPF pour ne pas dégrader leur réputation,
et parce que la plupart des gens veulent pouvoir émettre vers les Big Mailer Corps,
la plupart des émetteurs légitimes ne devraient pas rentrer dans ce cas.
Ne devraient rentrer dans ce cas que des petits émetteurs qui n'émettent pas depuis un grand nombre d'adresse,
et dont le greylisting serait identique qu'il soit par adresse IP ou domaine.

Ensuite,
nous avons les MX **qui publient des enregistrements SPF**,
englobant l'ensemble des Big Mailer Corps pour des raisons évidentes (difficile d'imposer SPF à d'autres sans se l'imposer à soi-même).
Pour ceux-ci,
plutôt que d'avoir recours a un greylisting par IP source,
on procède à un **greylisting SPF du domaine**.

En supposant que ce soit le premier email que nous recevons d'un MX,
quand une nouvelle connexion arrive,
le filtre garde trace de l'adresse IP source.
La session SMTP progresse jusqu'à la création d'une transaction SMTP et le client fournit l'émetteur d'envelope (MAIL FROM) :

```
S: 220 in.mailbrix.mx ESMTP OpenSMTPD
C: EHLO localhost
S: [...]
S: 250 in.mailbrix.mx Hello localhost [127.0.0.1], pleased to meet you
C: MAIL FROM:<gilles@poolp.org>
```

À ce stade,
le filtre peut faire une recherche SPF pour le domaine de l'émetteur d'enveloppe,
`poolp.org`.
S'il **ne trouve pas d'enregistrement SPF**,
il pourra **garder trace de l'adresse IP source** et demander une retransmission.

S'il **trouve un enregistrement SPF**,
il **vérifie que l'adresse IP source est valide pour cet enregistrement**.
Si elle ne l'est pas,
il peut supposer qu'elle ne provient pas du domaine émetteur,
le filtre va là encore procéder à un greylisting **par adresse IP source**.
Une approche plus stricte pourrait être de rejeter la session,
mais ce n'est pas le but d'un filtre de greylisting de décider si la violation SPF est acceptable ou non.

Si en revanche **l'adresse IP source est valide pour l'enregistrement SPF**,
au lieu de garder trace de l'adresse IP source,
le filtre **garde trace du domaine** et demande une retransmission.

Ignorons dorénavant le cas du greylisting par IP source pour nous concentrer sur le greylisting par domaine.

Lors d'une seconde connexion en provenance d'un domaine dont le filtre avait gardé trace,
si l'adresse IP source est valide pour l'enregistrement SPF,
le filtre va pouvoir valider que le domaine était déjà en demande de retransmission depuis une autre addresse IP et considérer le test comme réussi.
En **d'autres termes plus simples**,
si Gmail vient d'une adresse IP,
il est autorisé à venir d'une autre addresse IP **dès lors que les deux sont déclarées dans l'enregistrement SPF**.

Cette résolution SPF en live permet de garantir qu'il n'y a pas besoin de whitelister au préalable les adresses IP.
À la place il est possible de whitelister des domaines indépendamment des adresses IP avec lesquelles ils nous contacterons.
Cette façon de fonctionner est de plus compatible avec les domaines qui ne fournissent pas d'enregistrement SPF,
et qui dégradent en un greylisting par IP source.

Il n'est pas possible d'implémenter cela dans un daemon comme `spamd(8)`,
la décision de greylister ou de laisser la session progresser est prise au moment du `MAIL FROM`,
là ou la redirection `spamd(8)` est décidée par le firewall à la connexion.


Est-ce que ça peut être amélioré ?
--
Je ne sais pas trop mais je ne pense pas que l'on puisse radicalement améliorer l'idée.

Il y a surement quelques petites améliorations à faire à la marge,
mais je ne pense pas que l'on puisse faire de grosses grosses avancées :
un SPF-aware greylisting résout le problème du greylisting chez les Big Mailer Corps, ni plus, ni moins.
Aujourd'hui avec ce filtre,
je ne vois aucune différence de traitement entre Gmail et le petit domaine du coin.
Les deux passent le greylisting assez rapidement et sans que l'on ne se rende vraiment compte qu'il y a eu retransmission.

En gardant en tête que le spam est une industrie du volume,
je pense que du travail plus intéressant peut être fait en augmentant le coût de l'envoi de masse :
de la même manière que les spammeurs n'aiment pas la retransmission,
ils n'aiment **PAS DU TOUT** les hôtes lents parce qu'ils ont un impact **ÉNORME** sur la capacité à distribuer et sur la taille de la queue.
J'ai déjà implémenté des fonctionnalités dans ce sens au sein d'autres filtres en corrélant les temps de réponse à la réputation d'un domaine.

Si je passe plus de temps à essayer de faire d'OpenSMTPD une cible difficile pour les spammeurs,
il est certain que mes prochains travaux seront dans la lignée des sanctions par délais.

