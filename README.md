# Proposition GSoC 2024: Amélioration de `bulk_create` dans Django

## Introduction
Cette proposition vise à étendre la fonction bulk_create de Django pour supporter explicitement le passage de noms de contraintes uniques, y compris pour les cas où ces contraintes concernent des colonnes calculées ou des expressions. Ce besoin émerge du ticket #34943 du système de suivi de Django, révélant une limitation significative dans la gestion des conflits d'insertion en masse dans des scénarios complexes.

## Objectifs
- Développer une nouvelle fonctionnalité dans bulk_create permettant de spécifier des contraintes uniques par leurs noms, offrant ainsi une gestion plus granulaire des conflits d'insertion.
- Garantir la prise en charge des expressions et des colonnes calculées dans les contraintes uniques, permettant à Django de gérer des cas d'utilisation avancés.
- Élaborer une suite de tests exhaustifs et une documentation détaillée pour assurer l'adoption et la compréhension aisées de la nouvelle fonctionnalité.


## Plan de Projet

### Phase 1 : Recherche et Analyse
- **Analyse des Mécanismes Actuels :** Examen approfondi de la fonction bulk_create pour identifier les limitations existantes, notamment en ce qui concerne la gestion des contraintes uniques et les cas impliquant des colonnes calculées ou des expressions.
- **Benchmarking :** Étude des meilleures pratiques et des solutions existantes dans d'autres frameworks et langages de programmation pour gérer des situations similaires, afin d'inspirer notre approche.

### Phase 2 : Conception et Développement Initial
- **Dépréciation de `unique_fields` :** Comme suggéré initialement, déprécier l'utilisation de `unique_fields` en faveur de `unique_constraints`, qui peut être plus expressif et flexible.


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

- **Adaptation des fonction on_conflict_suffix_sql**: La fonction `on_conflict_suffix_sql` doit être adaptée pour interpréter `unique_constraints` et générer une clause `ON CONFLICT` adaptée en fonction des champs et des expressions impliqués dans les contraintes référencées.

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

## Calendrier
- **Semaines 1-3 :** Recherche et analyse des besoins.
- **Semaines 4-7 :** Conception et développement initial.
- **Semaines 8-9 :** Tests intensifs et début de la documentation.
- **Semaines 10-12 :** Finalisation de la documentation, révisions et préparation pour la soumission finale.

## À propos de Moi
Je suis [Votre Nom], étudiant(e) en [Votre Domaine d'Étude] à [Votre Université]. Passionné(e) par le développement web et contribuant activement à des projets open-source, je suis particulièrement intéressé(e) par l'amélioration de frameworks comme Django pour faciliter et optimiser le développement web.

## Communication
Je prévois de communiquer régulièrement avec mon mentor et la communauté à travers le forum Django, les pull requests et les issues sur GitHub, en m'assurant que le projet reste aligné avec les attentes et bénéficie d'une collaboration étroite.

## Conclusion
En permettant à bulk_create de gérer de manière plus sophistiquée les contraintes uniques, cette proposition vise à enrichir significativement Django, en facilitant le développement d'applications robustes et performantes. J'aspire à contribuer à cette avancée, en apportant mon énergie et mon expertise au sein de la vibrante communauté Django.

