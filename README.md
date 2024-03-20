# GSoC 2024 Proposal: Significant Enhancement of Django's `bulk_create`

## Introduction
This proposal aims to significantly enhance Django's `bulk_create` function by integrating two major improvements. The first part of this proposal focuses on finalizing and integrating the new feature initiated in ticket [#34277](https://code.djangoproject.com/ticket/34277), which allows adding a WHERE clause to `QuerySet.bulk_create()` when using `update_conflicts=True`. The second part addresses ticket [#34943](https://code.djangoproject.com/ticket/34943), aiming to extend `bulk_create` to explicitly support passing unique constraint names, including cases involving computed columns or expressions. This integrated approach will resolve significant limitations in bulk insertion conflict management, making `bulk_create` more flexible and powerful for complex usage scenarios.

## Objectives
- **[Finalize](https://github.com/django/django/pull/17515) and Integrate the Feature from Ticket [#34277](https://code.djangoproject.com/ticket/34277):** Ensure the successful completion and integration of the new feature allowing the use of a WHERE clause in `bulk_create` for conflict updates, ensuring this enhancement is well documented and tested.
- **Develop the Feature from Ticket [#34943](https://code.djangoproject.com/ticket/34943):** Extend `bulk_create` to allow specifying unique constraints by their names, thus facilitating conflict management for advanced cases involving computed columns or expressions.
- **Ensure Comprehensive Test Coverage:** Develop a complete suite of tests for each of the developed features to ensure their reliability and integrity across various use cases.
- **Provide Clear and Comprehensive Documentation:** Ensure the new features are accompanied by detailed documentation, including usage examples and migration guides for developers.
- **Continue [Work](https://github.com/django/django/pull/17754) on Ticket [#28821](https://code.djangoproject.com/ticket/28821):** If the main objectives for tickets [#34277](https://code.djangoproject.com/ticket/34277) and [#34943](https://code.djangoproject.com/ticket/34943) are met within the allotted time, additional work on ticket #28821 could be considered as a bonus. This ticket explores using `bulk_create` with multi-table inheritance and will be addressed based on progress and remaining time, not being an integral part of the GSoC commitments.

This proposal adopts a strategic and integrated approach to improve the `bulk_create` function, addressing key limitations and providing new capabilities that will enrich the Django ecosystem. By prioritizing the completion of ongoing work on ticket [#34277](https://code.djangoproject.com/ticket/34277), this proposal aims to bring tangible improvements to the Django community rapidly while laying the groundwork for more ambitious future enhancements.

## Project Plan

### Phase 1: Research and Analysis
- **Current Mechanisms Analysis:** Conduct a thorough examination of the `bulk_create` function to identify existing limitations, particularly regarding unique constraints management and cases involving computed columns or expressions.
- **Benchmarking:** Study best practices and existing solutions in other frameworks and programming languages for handling similar situations to inspire our approach.

### Phase 2: Initial Design and Development
- **Deprecating `unique_fields`:** As initially suggested by Simon Charette:
>My recommendation would be to introduce a unique_constraint: str | tuple[str | Expression] kwarg and deprecate unique_fields. When provided a str it would be a reference to a UniqueConstraint by .name and when it's a tuple it would be expected to be an index expression where str are resolved to field names.

Deprecate the use of `unique_fields` in favor of `unique_constraints`, which can be more expressive and flexible. It could accept not only the names of unique constraints defined in `Meta.constraints` but also potentially tuples representing groups of fields that must be unique together.

- **Using `unique_constraints`:** Allow `unique_constraints` to accept constraint names (as strings) defined in the model's `Meta` class. For complex expressions, instead of resolving them directly in the SQL clause, Django could resolve them upfront as a set of fields or expressions representing the constraint, using index inference mechanisms or constraint metadata.

- **Dynamically Generating the SQL Clause:** When executing `bulk_create`, Django would dynamically generate the appropriate SQL clause (e.g., `ON CONFLICT(...)`) based on the fields or expressions provided by `unique_constraints`. This would avoid relying on the `ON CONFLICT ON CONSTRAINT` clause and instead use index inference or an explicit list of fields/expressions involved in the constraint.

**Conceptual Example:**

```python
class UserTopic(models.Model):
    # Model fields
    class Meta:
        constraints = [
            UniqueConstraint(fields=['user_profile_id', 'stream_id', Upper('topic_name')], name='unique_profile_stream_topic')
        ]

# Usage in bulk_create
UserTopic.objects.bulk_create(
    [ut],
    update_fields=['last_updated', 'visibility_policy'],
    unique_constraints=['unique_profile_stream_topic']  # Reference to the constraint defined in Meta
)
```

In this example, instead of directly using `ON CONFLICT ON CONSTRAINT`, Django could interpret `unique_profile_stream_topic` to generate an `ON CONFLICT(...)` clause tailored to the fields and expressions defined in the `UniqueConstraint` of the model.

This approach attempts to balance flexibility, robustness, and adherence to PostgreSQL recommendations while providing a clear and expressive interface for Django developers.

- **Adapting the `on_conflict_suffix_sql` Function:** The `on_conflict_suffix_sql` function needs to be adapted to interpret `unique_constraints` and generate an `ON CONFLICT` clause based on the fields and expressions involved in the referenced constraints.

Suppose `unique_constraints` can be a list containing either constraint names (strings) defined in Meta.constraints or tuples of fields that must be unique together. To simplify, we will not directly handle complex expressions like `Upper('topic_name')` here, but we will assume a proper unique constraint has already been defined to cover this case.

This code is specific to the PostgreSQL backend, located in **django/db/backends/postgresql/operations.py**
```python
def on_conflict_suffix_sql(self, on_conflict, unique_constraints):
    if on_conflict == OnConflict.IGNORE:
        return "ON CONFLICT DO NOTHING"
    elif on_conflict == OnConflict.UPDATE and unique_constraints:
        # Initialize the parts of the ON CONFLICT clause
        on_conflict_fields = []

        for constraint in unique_constraints:
            if isinstance(constraint, str):  # If the constraint is a string, it's the name of a constraint in Meta.constraints
                # Resolve the fields involved in this named constraint
                fields = self.resolve_constraint_to_fields(constraint)
                on_conflict_fields.extend(fields)
            elif isinstance(constraint, (list, tuple)):  # If the constraint is a list or tuple, it's a set of fields
                on_conflict_fields.extend(constraint)

        # Create the ON CONFLICT part of the SQL clause using the resolved fields
        conflict_fields_sql = ", ".join([self.quote_name(field) for field in on_conflict_fields])
        return f"ON CONFLICT ({conflict_fields_sql}) DO UPDATE SET ..."

    # Fallback if none of the above conditions are met
    return super().on_conflict_suffix_sql(on_conflict, unique_constraints)
```

- **Resolving Named Constraints to Fields:**
The `resolve_constraint_to_fields` method (mentioned in the example but not defined) would be responsible for taking the name of a constraint and determining which fields are involved. This resolution would likely require introspecting the Django model to find the correspondence between constraint names and involved fields. This part of the process could be complex and model-specific, thus requiring careful implementation.
```python
def resolve_constraint_to_fields(self, constraint_name):
    # Pseudocode example for resolving fields involved in a named constraint
    # This is greatly simplified and should be adapted based on your model's actual structure and constraints
    model_meta = self.model._meta
    for constraint in model_meta.constraints:
        if constraint.name == constraint_name:
            return constraint.fields  # Simplistic assumption that the constraint object has a 'fields' attribute
    return []
```
This approach assumes some introspection capability and a clear convention for defining constraints in Django models, allowing for the dynamic resolution of fields involved in constraints at runtime. It aims to provide flexibility in specifying unique constraints in `bulk_create`, without relying directly on database-specific SQL syntax, while adhering to PostgreSQL recommendations and other database management systems' standards.

- **Developing Usage Examples:** Illustrate the use of the extended functionality in real-world use cases.

```python
UserTopic.objects.bulk_create([...], unique_constraints=['constraint_name', ...])
```

- **Initial Tests:** Writing unit tests to validate basic functionalities and ensure no regressions.
- **Initial Documentation:** Drafting initial documentation, including the new API description and usage examples.

### Phase 3: In-depth Testing, Final Documentation, and Revisions

- **In-depth Testing:** Developing a more comprehensive test suite to cover a wide range of scenarios, including edge cases and tests specific to different database engines supported by Django.

```python
def test_bulk_create_with_unique_constraints(self):
    # Testing the management of unique constraints in bulk_create
    # Implementation...
```

- **Finalizing Documentation:** Developing detailed and instructive documentation that incorporates usage examples, best practices, and tips for developers. The documentation will also include information on how to contribute to and extend the functionality.
- **Feedback and Adjustments:** Regular interaction with Django mentors and the community to gather feedback on the proposed implementation and make necessary adjustments to align as closely as possible with community standards and expectations.
- **Preparation for Merging:** Once all tests have passed and community feedback has been integrated, preparing the final pull request for integration into Django's main codebase.

## Project Plan Flexibility
I acknowledge that software development is dynamic and can unveil unexpected challenges. Therefore, although I present an initial approach here, I am prepared to adjust my plan based on the realities encountered and feedback from the Django community. This adaptability is crucial for effectively addressing project needs and contributing constructively.

## Timeline
- **Weeks 1-3:** Research and needs analysis.
- **Weeks 4-7:** Initial design and development.
- **Weeks 8-9:** Intensive testing and documentation start.
- **Weeks 10-12:** Finalization of documentation, revisions, and preparation for final submission.

## Definition of Success
The success of this GSoC project will be measured by several key milestones related to the progress and completion of the tickets I am working on:

- **Initial Success: Finalization and Merging of Ticket #34277** - The first success indicator will be the completion of work on ticket #34277, including its review by the community, approval, and merging into Django's main branch. This will represent a tangible improvement to the bulk_create functionality, thereby enriching the Django framework.

- **Intermediate Success: Progress on Ticket #34943** - The second level of success will be marked by significant progress on ticket #34943, aiming to develop and test the new feature that enables bulk_create to handle complex unique constraints. Intermediate success will include the submission of a pull request for review.

- **Final Success: Contribution to Ticket #28821** - As a bonus objective, final success will aim to make a substantial contribution to ticket #28821. Although this ticket is not the main priority of the GSoC, any progress made on this front will be considered an additional success, illustrating ongoing contributions to Django's improvement.

## About Me
I am [ISSAKA HAMA Barhamou](https://hamabarhamou.onrender.com/), holding a degree in Mathematics and Computer Science from the [University of Abdou Moumouni](https://fr.wikipedia.org/wiki/Universit%C3%A9_Abdou-Moumouni) in Niamey/[Niger](https://fr.wikipedia.org/wiki/Niger). I have also earned a software engineering certification, specializing in Backend, through the [ALX AFRICA](https://www.alxafrica.com/) program.

My passion for problem-solving has led me to actively engage on various coding platforms, where I enjoy taking on diverse challenges:

- [CodinGame](https://www.codingame.com/profile/4b64838485f1e54cce2d616e201bb7969377233)
- [France-IOI](http://www.france-ioi.org/user/perso.php?sLogin=barhamou)
- [LeetCode](https://leetcode.com/barhamou/)
- [HackerRank](https://www.hackerrank.com/profile/hamabarhamou)

In addition to programming, I am also passionate about cybersecurity, a field in which I continue to develop my skills through platforms such as:

- [TryHackMe]()
- [Root-Me]()

You can learn more about my journey and projects through my [portfolio](https://hamabarhamou.onrender.com/) and my [GitHub](https://github.com/HamaBarhamou).

I am also active on social media, where I regularly share my experiences and discoveries:

- [Mastodon](https://mastodon.social/@HamaBarhamou)
- [Twitter](https://twitter.com/hama_barhamou)
- [LinkedIn](https://www.linkedin.com/in/barhamou-hama-90047b179/)

In terms of Django development, I am currently working on two significant tickets:

- Ticket [#34277](https://code.djangoproject.com/ticket/34277) with [the corresponding pull request](https://github.com/django/django/pull/17515)
- Ticket [#28821](https://code.djangoproject.com/ticket/28821) and its [pull request](https://github.com/django/django/pull/17754)

These works illustrate my commitment to improving the Django framework and sharing innovative solutions with the community.

Looking forward to connecting with you!

## Communication
I plan to communicate regularly with my mentor and the Django community through the Django forum, pull requests, and issues on GitHub, ensuring the project remains aligned with expectations and benefits from close collaboration.

## Conclusion
This proposal outlines my current work plan on key enhancements to Django's `bulk_create` function, through tickets [#34277](https://code.djangoproject.com/ticket/34277), [#34943](https://code.djangoproject.com/ticket/34943), and [#28821](https://code.djangoproject.com/ticket/28821). My commitment to these tickets is independent of the acceptance of this proposal for the Google Summer of Code. I am determined to make these significant contributions to Django, leveraging this opportunity to align my efforts with the GSoC objectives, thereby creating a synergy between my ongoing work and the collaborative and innovative spirit of the program. By participating in GSoC, I aim not only to advance on these already accepted tickets but also to share this experience with the community, enhance my knowledge, and hopefully accelerate the development of these important features for Django. It is with this spirit of collaboration and continuous innovation that I present my proposal, hoping to bring tangible value to Django and its vibrant developer community.