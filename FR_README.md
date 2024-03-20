# Proposition GSoC 2024: Amélioration significative de `bulk_create` dans Django

## Introduction
Cette proposition vise à améliorer de manière significative la fonction `bulk_create` de Django en intégrant deux améliorations majeures. La première partie de cette proposition se concentre sur la finalisation et l'intégration de la nouvelle fonctionnalité initiée dans le ticket [#34277](https://code.djangoproject.com/ticket/34277), qui permet d'ajouter une clause WHERE à `QuerySet.bulk_create()` lors de l'utilisation de `update_conflicts=True`. La seconde partie de la proposition aborde le ticket [#34943](https://code.djangoproject.com/ticket/34943), qui vise à étendre `bulk_create` pour supporter explicitement le passage de noms de contraintes uniques, y compris pour les cas où ces contraintes impliquent des colonnes calculées ou des expressions. Cette approche intégrée permettra de résoudre des limitations significatives dans la gestion des conflits d'insertion en masse et de rendre bulk_create plus flexible et puissant pour des scénarios d'utilisation complexes.

## Objectifs
- **Finaliser ([PR](https://github.com/django/django/pull/17515)) et Intégrer la Fonctionnalité du Ticket [#34277](https://code.djangoproject.com/ticket/34277) :** Assurer l'achèvement et l'intégration réussie de la nouvelle fonctionnalité permettant l'utilisation d'une clause WHERE dans bulk_create lors de la mise à jour des conflits, en veillant à ce que cette amélioration soit bien documentée et testée.
- **Développer la Fonctionnalité du Ticket [#34943](https://code.djangoproject.com/ticket/34943) :** Étendre bulk_create pour permettre la spécification de contraintes uniques par leurs noms, facilitant ainsi la gestion des conflits d'insertion pour des cas avancés impliquant des colonnes calculées ou des expressions.
- **Garantir une Couverture de Test Exhaustive :** Pour chacune des fonctionnalités développées, élaborer une suite de tests complète pour garantir leur fiabilité et leur intégrité dans divers scénarios d'utilisation.
- **Fournir une Documentation Claire et Complète :** Assurer que les nouvelles fonctionnalités sont accompagnées d'une documentation détaillée, incluant des exemples d'utilisation et des guides de migration pour les développeurs.
- **Poursuite des [Travaux](https://github.com/django/django/pull/17754) sur le Ticket #28821:** Si les objectifs principaux concernant les tickets #34277 et #34943 sont atteints dans les délais impartis, un travail additionnel sur le ticket #28821 pourrait être envisagé comme un bonus. Ce ticket explore l'utilisation de bulk_create avec l'héritage multi-table et sera abordé en fonction des progrès réalisés et du temps restant, sans faire partie intégrante des engagements du GSoC.

Cette proposition adopte une approche stratégique et intégrée pour améliorer la fonction bulk_create, en s'attaquant à des limitations clés et en offrant de nouvelles capacités qui enrichiront l'écosystème Django. En priorisant la finalisation du travail en cours sur le ticket #34277, cette proposition vise à apporter rapidement des améliorations tangibles à la communauté Django, tout en posant les bases pour des améliorations futures plus ambitieuses.


## Plan de Projet

### Phase 1 : Recherche et Analyse
- **Analyse des Mécanismes Actuels :** Examen approfondi de la fonction bulk_create pour identifier les limitations existantes, notamment en ce qui concerne la gestion des contraintes uniques et les cas impliquant des colonnes calculées ou des expressions.
- **Benchmarking :** Étude des meilleures pratiques et des solutions existantes dans d'autres frameworks et langages de programmation pour gérer des situations similaires, afin d'inspirer notre approche.

### Phase 2 : Conception et Développement Initial
- **Dépréciation de `unique_fields` :** Comme suggéré initialement, déprécier l'utilisation de `unique_fields` en faveur de `unique_constraints`, qui peut être plus expressif et flexible. Elle peut accepter non seulement des noms de contraintes uniques définies dans `Meta.constraints`, mais aussi potentiellement des tuples qui représentent des groupes de champs devant être uniques ensemble.


- **Utilisation de `unique_constraints` :** Permettre à `unique_constraints` d'accepter des noms de contraintes (comme des chaînes) qui sont définies dans la classe `Meta` du modèle. Pour les expressions complexes, au lieu de les résoudre directement dans la clause SQL, Django pourrait les résoudre en amont en tant qu'ensemble de champs ou d'expressions représentant la contrainte, en utilisant les mécanismes d'inférence d'index ou les métadonnées de contrainte disponibles.

- **Génération Dynamique de la Clause SQL :** Lors de l'exécution de `bulk_create`, Django générerait dynamiquement la clause SQL appropriée (par exemple, `ON CONFLICT(...)` avec les champs ou expressions appropriés), en s'appuyant sur les informations fournies par unique_constraints. Cela éviterait de s'appuyer sur la clause `ON CONFLICT ON CONSTRAINT` et utiliserait plutôt l'inférence d'index ou une liste explicite des champs/expression impliqués dans la contrainte.

**Exemple Conceptuel :**
```
class UserTopic(models.Model):
    # Champs du modèle
    class Meta:
        constraints = [
            UniqueConstraint(fields=['user_profile_id', 'stream_id', Upper('topic_name')], name='unique_profile_stream_topic')
        ]

# Utilisation dans bulk_create
UserTopic.objects.bulk_create(
    [ut],
    update_fields=['last_updated', 'visibility_policy'],
    unique_constraints=['unique_profile_stream_topic']  # Référence au nom de la contrainte définie dans Meta
)
```

Dans cet exemple, au lieu d'utiliser directement `ON CONFLICT ON CONSTRAINT`, Django pourrait interpréter `unique_profile_stream_topic` pour générer une clause `ON CONFLICT(...)` adaptée aux champs et expressions définis dans la contrainte `UniqueConstraint` du modèle.

Cette approche tente de concilier flexibilité, robustesse et respect des recommandations PostgreSQL, tout en offrant une interface claire et expressive pour les développeurs Django.

- **Adaptation des fonction `on_conflict_suffix_sql`**: La fonction `on_conflict_suffix_sql` doit être adaptée pour interpréter `unique_constraints` et générer une clause `ON CONFLICT` adaptée en fonction des champs et des expressions impliqués dans les contraintes référencées.

Supposons que unique_constraints peut être une liste contenant soit des noms de contraintes (chaînes) définies dans Meta.constraints, soit des tuples de champs qui doivent être uniques ensemble. Pour simplifier, nous ne traiterons pas directement les expressions complexes comme `Upper('topic_name')` ici, mais nous supposerons qu'une contrainte unique appropriée a déjà été définie pour couvrir ce cas.

Ce code est spécifique au backend PostgreSQL, situé dans **django/db/backends/postgresql/operations.py**
```
def on_conflict_suffix_sql(self, on_conflict, unique_constraints):
    if on_conflict == OnConflict.IGNORE:
        return "ON CONFLICT DO NOTHING"
    elif on_conflict == OnConflict.UPDATE and unique_constraints:
        # Initialiser les parties de la clause ON CONFLICT
        on_conflict_fields = []

        for constraint in unique_constraints:
            if isinstance(constraint, str):  # Si la contrainte est une chaîne, c'est le nom d'une contrainte dans Meta.constraints
                # Résoudre les champs impliqués dans cette contrainte nommée
                fields = self.resolve_constraint_to_fields(constraint)
                on_conflict_fields.extend(fields)
            elif isinstance(constraint, (list, tuple)):  # Si la contrainte est une liste ou un tuple, c'est un ensemble de champs
                on_conflict_fields.extend(constraint)

        # Créer la partie ON CONFLICT de la clause SQL en utilisant les champs résolus
        conflict_fields_sql = ", ".join([self.quote_name(field) for field in on_conflict_fields])
        return f"ON CONFLICT ({conflict_fields_sql}) DO UPDATE SET ..."

    # Fallback si aucune des conditions ci-dessus n'est remplie
    return super().on_conflict_suffix_sql(on_conflict, unique_constraints)

```

- **Résolution des Contraintes Nominales en Champs:**
La méthode `resolve_constraint_to_fields` (mentionnée dans l'exemple mais non définie) aurait pour responsabilité de prendre le nom d'une contrainte et de déterminer quels champs sont impliqués. Cette résolution nécessiterait probablement une introspection du modèle Django pour trouver la correspondance entre les noms de contraintes et les champs impliqués. Cette partie du processus pourrait être complexe et dépendante du modèle spécifique, et donc nécessiterait une implémentation soignée.
```
def resolve_constraint_to_fields(self, constraint_name):
    # Exemple de pseudocode pour résoudre les champs impliqués dans une contrainte nommée
    # Ceci est fortement simplifié et devrait être adapté en fonction de la structure réelle de votre modèle et des contraintes
    model_meta = self.model._meta
    for constraint in model_meta.constraints:
        if constraint.name == constraint_name:
            return constraint.fields  # Supposition simpliste que l'objet constraint a un attribut 'fields'
    return []
```
Cette approche suppose une certaine capacité d'introspection et une convention claire pour la définition des contraintes dans les modèles Django, permettant ainsi une résolution dynamique des champs impliqués dans les contraintes au moment de l'exécution. Elle vise à offrir une flexibilité dans la spécification des contraintes uniques dans `bulk_create`, sans s'appuyer directement sur la syntaxe SQL spécifique à la base de données, et tout en respectant les recommandations de PostgreSQL et d'autres systèmes de gestion de base de données.

- **Développement d'Exemples d'Utilisation :** Illustration de l'utilisation de la fonctionnalité étendue dans des cas d'usage réels.

```
UserTopic.objects.bulk_create([...], unique_constraints=['constraint_name', ...])
```

- **Tests Initiaux :** Écriture de tests unitaires pour valider les fonctionnalités de base et s'assurer de l'absence de régressions.
- **Documentation Initiale :** Rédaction des premières ébauches de documentation, y compris la description de la nouvelle API et des exemples d'utilisation.

### Phase 3 : Tests Approfondis, Documentation Finale et Révisions

- **Tests Approfondis :** Développement d'une suite de tests plus complète pour couvrir une gamme étendue de scénarios, y compris des cas limites et des tests spécifiques aux différents moteurs de base de données supportés par Django.
python

```
def test_bulk_create_with_unique_constraints(self):
    # Test de la gestion des contraintes uniques dans bulk_create
    # Implémentation ...
```

- **Finalisation de la Documentation :** Élaboration d'une documentation détaillée et instructive, intégrant des exemples d'utilisation, des meilleures pratiques et des conseils pour les développeurs. La documentation inclura également des informations sur la manière de contribuer et d'étendre la fonctionnalité.
- **Feedback et Ajustements :** Interaction régulière avec les mentors et la communauté Django pour recueillir des retours sur l'implémentation proposée, et réalisation des ajustements nécessaires pour s'aligner au mieux avec les standards et les attentes de la communauté.
- **Préparation pour la Fusion :** Une fois tous les tests passés et les retours communautaires intégrés, préparation de la pull request finale pour l'intégration dans le code source de Django.

## Flexibilité du Plan de Projet
Je reconnais que le développement logiciel est dynamique et peut révéler des défis inattendus. Ainsi, bien que je présente ici une approche initiale, je suis prêt à ajuster mon plan en fonction des réalités rencontrées et des retours de la communauté Django. Cette adaptabilité est essentielle pour répondre efficacement aux besoins du projet et contribuer de manière constructive.

## Calendrier
- **Semaines 1-3 :** Recherche et analyse des besoins.
- **Semaines 4-7 :** Conception et développement initial.
- **Semaines 8-9 :** Tests intensifs et début de la documentation.
- **Semaines 10-12 :** Finalisation de la documentation, révisions et préparation pour la soumission finale.

## Définition du Succès
Le succès de ce projet GSoC sera mesuré par plusieurs jalons clés correspondant à l'avancement et à la finalisation des tickets sur lesquels je travaille :

- **Succès Initial : Finalisation et Fusion du Ticket #34277** - Le premier indicateur de succès sera la finalisation des travaux sur le ticket #34277, incluant sa révision par la communauté, son approbation, et sa fusion dans la branche principale de Django. Cela représentera une amélioration tangible de la fonctionnalité bulk_create, enrichissant ainsi le framework Django.

- **Succès Intermédiaire : Avancement sur le Ticket #34943** - Le second niveau de succès sera marqué par des progrès significatifs sur le ticket #34943, avec l'objectif de développer et de tester la nouvelle fonctionnalité permettant à bulk_create de gérer des contraintes uniques complexes. La réussite intermédiaire inclura la soumission d'une pull request pour révision.

- **Succès Final : Contribution au Ticket #28821** - En tant qu'objectif bonus, le succès final visera à apporter une contribution substantielle au ticket #28821. Bien que ce ticket ne soit pas la priorité principale du GSoC, tout progrès réalisé sur ce front sera considéré comme un succès supplémentaire, illustrant une contribution continue à l'amélioration de Django.

## À propos de Moi
Je m'appelle ISSAKA HAMA Barhamou, titulaire d'une licence en Mathématiques et Informatique de l'Université Abdou Moumouni de Niamey. J'ai également obtenu une certification en génie logiciel, spécialité Backend, grâce au programme ALX AFRICA.

Ma passion pour la résolution de problèmes m'a conduit à m'engager activement sur plusieurs plateformes de codage, où je me plais à relever des défis variés :

- [CodinGame](https://www.codingame.com/profile/4b64838485f1e54cce2d616e201bb7969377233)
- [France-IOI](http://www.france-ioi.org/user/perso.php?sLogin=barhamou)
- [LeetCode](https://leetcode.com/barhamou/)
- [HackerRank](https://www.hackerrank.com/profile/hamabarhamou)

Outre la programmation, je suis également passionné par la cybersécurité, un domaine où je continue à développer mes compétences à travers des plateformes telles que :

- [TryHackMe]()
- [Root-Me]()

Vous pouvez découvrir davantage sur mon parcours et mes projets via mon [portfolio](https://hamabarhamou.onrender.com/) et mon [github](https://github.com/HamaBarhamou).

Je suis également actif sur les réseaux sociaux, où je partage régulièrement mes expériences et découvertes :

- [Mastodon](https://mastodon.social/@HamaBarhamou)
- [Twitter](https://twitter.com/hama_barhamou)
- [LinkedIn](https://www.linkedin.com/in/barhamou-hama-90047b179/)

En matière de développement Django, je suis actuellement impliqué dans le travail sur deux tickets significatifs :

- Ticket [#34277](https://code.djangoproject.com/ticket/34277) avec [la pull request correspondante](https://github.com/django/django/pull/17515)
- Ticket [#28821](https://code.djangoproject.com/ticket/28821) et sa [pull request](https://github.com/django/django/pull/17754)
Ces contributions illustrent mon engagement envers l'amélioration de frameworks open-source et le partage de solutions innovantes avec la communauté.

Au plaisir d'échanger avec vous !

## Communication
Je prévois de communiquer régulièrement avec mon mentor et la communauté à travers le forum Django, les pull requests et les issues sur GitHub, en m'assurant que le projet reste aligné avec les attentes et bénéficie d'une collaboration étroite.


## Conclusion
Cette proposition décrit mon plan de travail actuel sur des améliorations clés de la fonction `bulk_create` dans Django, à travers les tickets #34277, #34943, et #28821. Mon engagement sur ces tickets est indépendant de l'acceptation de cette proposition pour le Google Summer of Code. Je suis déterminé à apporter ces contributions significatives à Django, en tirant parti de cette opportunité pour aligner mes efforts avec les objectifs du GSoC, réalisant ainsi une synergie entre mes travaux en cours et l'esprit collaboratif et innovant du programme. En participant au GSoC, je souhaite non seulement avancer sur ces tickets déjà acceptés mais aussi partager cette expérience avec la communauté, enrichir mes connaissances et, espérons-le, accélérer le développement de ces fonctionnalités importantes pour Django. C'est dans cet esprit de collaboration et d'innovation continue que je présente ma proposition, dans l'espoir d'apporter une valeur ajoutée tangible à Django et à sa vibrante communauté de développeurs.
