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
- **Extension de l'API bulk_create :** Adapter bulk_create pour deprecier le paramètre unique_fields en unique_constraints qui peut inclure à la fois des noms de contraintes uniques et des expressions.
```
def bulk_create(self, objs, batch_size=None, ignore_conflicts=False, unique_constraints=None):
    # Implémentation ...
```

- **Développement de la Logique de Résolution de Contraintes :** Intégrer une méthode pour gérer à la fois les contraintes par nom et les expressions, en utilisant la distinction entre les types pour générer la clause SQL appropriée.

```
def handle_constraint(self, constraint):
    if isinstance(constraint, Expression):
        return self.resolve_expression_to_sql(constraint)
    return self.quote_name(constraint)
```

- **Adaptation des fonction on_conflict_suffix_sql**: Adaptation spécifique pour générer la clause ON CONFLICT, y compris la prise en charge de la clause ON CONFLICT ON CONSTRAINT pour les noms de contraintes et la gestion adaptée des expressions.

```
# Ce code est spécifique au backend PostgreSQL, situé dans django/db/backends/postgresql/operations.py

def on_conflict_suffix_sql(self, fields, on_conflict, unique_constraints):
    if on_conflict == OnConflict.IGNORE:
        return "ON CONFLICT DO NOTHING"
    elif on_conflict == OnConflict.UPDATE:
        # Utilisation de handle_constraint pour gérer les noms de contraintes et les expressions
        resolved_constraints = [self.handle_constraint(constraint) for constraint in unique_constraints]

        # Génération de la clause ON CONFLICT en fonction des contraintes résolues
        conflict_clause = ", ".join(resolved_constraints)
        
        # Générer la clause de mise à jour en utilisant les champs spécifiés dans 'fields'
        update_clause = ", ".join(
            "%s = EXCLUDED.%s" % (self.quote_name(field.name), self.quote_name(field.name)) for field in fields
        )

        return "ON CONFLICT(%s) DO UPDATE SET %s" % (conflict_clause, update_clause)

    # Si les conditions ci-dessus ne sont pas remplies, appeler la méthode parente
    return super().on_conflict_suffix_sql(fields, on_conflict, unique_constraints)
```

- 1- Gestion des Contraintes et Expressions : La méthode handle_constraint est utilisée pour chaque élément dans unique_constraints. Cette méthode est responsable de la distinction entre les contraintes nommées et les expressions, en retournant la représentation SQL appropriée pour chacune.

- 2- Clause ON CONFLICT : Les contraintes résolues par handle_constraint sont combinées pour former la clause ON CONFLICT. Pour les expressions complexes, handle_constraint devrait être en mesure de résoudre l'expression en une chaîne SQL valide. Pour les noms de contraintes, cela pourrait directement retourner "ON CONSTRAINT <nom_de_la_contrainte>".

- 3- Clause DO UPDATE SET : La clause de mise à jour est construite en utilisant les champs spécifiés dans fields, en s'assurant que chaque champ est mis à jour avec sa valeur correspondante dans la clause EXCLUDED.

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

