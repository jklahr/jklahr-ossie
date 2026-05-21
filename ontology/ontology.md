# OSI - Ontology Specification

**Version:** 0.1.1

## Table of Contents

1. [Enumerations](#enumerations)
2. [Ontology](#ontologies)
3. [Ontology map](#ontology-maps)
4. [Complete Example](#complete-example)

---

## Enumerations

Standard enumeration values used throughout the specification.

### Concept types

Ontologies distinguish two different kinds of concepts.
An entity type is a concept that represents real-world objects that must be referenced using other information.
For instance, a person might be referenced by their social security number
or private e-mail address.
A value type is a concept that represents instances of some data type
(i.e SQL types like Integer or String) with additional semantics.
For instance, a social-security number is a string or positive integer that comprises exactly nine digits.

| Multiplicity | Description |
|---------|-------------|
| `EntityType` | Real-world concept that must be referenced using other information |
| `ValueType` | A datatype with additional semantics |

### Multiplicities

The allowable multiplicities of relationships defined in the [Ontology](#ontology) section.

| Multiplicity | Description |
|---------|-------------|
| `ManyToOne` | The last role of a relationship is uniquely determined by the other roles |
| `OneToOne` | The relationship is ManyToOne in both directions (only for binary relationships) |

## Ontologies

Enterprise data are often modeled conceptualy in the form of an ontology with concepts, relationships,
and business rules. This section describes how to represent ontologies hierarchically as a
top-level collection of components, each of which defines some concept and the relationships
that pertain primarily to that concept.


### Concepts

Concepts represent the types of things that have meaning in a business setting, e.g., person, company,
or salary. Each concept is either an entity type or a value type. Ontologies implicitly include a
value-type for each basic data type, like `Integer`, `Decimal`, and `String`, and an entity type
called `Any`. Every other concept ontology extends (is a subtype of) one of these concepts.

Concepts conform to the following schema:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `concept` | string | Yes | Unique name of this concept |
| `type` | ConceptType | Yes | Entity type or value type |
| `description` | string | No | Human-readable description |
| `relationships` | list | No | Relationships where this concept plays the first role |
| `extends` | list | No | Names of this concept's supertypes |
| `derived_by` | list | No | Expressions that derive this concept's population |
| `identify_by` | list | No | Names of relationships that uniquely reference objects of this concept |
| `requires` | list | No | Expressions that constrain this concept's population |

### Extends

Every user-declared concept extends one or more concepts in the ontology. The new concept
is a sutype of each concept that it extends, and the extended concepts are its supertypes.

Any concept that directly or indirectly extends a value type like `Integer` or `String` is a value type.
Any concept that does not extend some value type is an entity type, and if a concept declares no extends
list, then it is assumed to extend the built-in entity type `Any`. If `SocialSecurityNr` extends `Integer`
and `Employee` extends `Person`, which declares no extends list, then `SocialSecurityNr` is a value type
and both `Person` and `Employee` are entity types.

### Relationships

Relationships relate objects of one or more concepts and declare how to verbalize links among
those objects. Relationships have set (as opposed to bag) semantics, and links do not contain
nulls.

Each relationship that is declared under a concept conforms to the following schema:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Part of the identifier for this relationship |
| `description` | string | No | Human-readable description |
| `multiplicity` | enum | No | Multiplicity constraint |
| `roles` | list | No | List of additional roles in this relationship |
| `derived_by` | list | No | Expressions that derive links of this relationship |
| `requires` | list | No | Expressions that constrain this relationship's population |
| `verbalizes` | list | Yes | Patterns describing how to verbalize links |

Each relationship is uniquely identified by a prepending its declared name with that of the containing
concept. For instance, in:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: Person
          type: EntityType
          identify_by: [ nr ]
        relationships:
          - name: nr
            roles:
              - player: SocialSecurityNr
            verbalizes: [ '{Person} is identified by {SocialSecurityNr}' ]
            multiplicity: OneToOne
          - name: earns
            roles:
              - player: Salary
            multiplicity: ManyToOne
            verbalizes: [ "{Person} earns {Salary}" ]
          ...
```

the relationship is identified by the string `Person.earns`. This convention naturally supports
expressions that navigate over the links of relationships using the “dot-join” operator in a
manner that is familiar to object-oriented programming languages. This relationship links
`Person` and `Salary` objects and verbalizes each link as “Person earns Salary.” 

#### Roles

Objects play roles in the links of a relationship. If you think of a relationship as a narrow table,
then its links are like rows and its roles are like columns. Each role is played by a concept that
constrains the type of objects that can play that role in any link. In `Person.earns`, `Person` and
`Salary` play the first and second roles respectively.

By convention, the first role of any relationship is played by the concept under which the
relationship is declared. Any additional roles are enumerated in order in the roles list
using this schema:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `player` | string | Yes | Name of the concept that plays this role |
| `name` | string | No | Optional role name |

For instance, in:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: Person
          type: EntityType
        relationships:
          - name: files_married_joint
            verbalizes: [ "{Person} files married filing joint" ]
          - name: purchased_on
            roles:
              - player: Vehicle
              - player: Date
            multiplicity: ManyToOne
            verbalizes: [ "{Person} puchased {Vehicle} on {Date}" ]
```

the unary relationship `Person.files_married_joint` has an empty roles list, while the
ternary relationship `Person.purchased_on` declares two additional roles played by
`Vehicle` and `Date` respectively,

The role player often suffices to distinguish the role within its relationship, but when
the same concept plays more than one role, the user must declare a distinguising name for
any additional role whose player does not suffice to distinguish it from other roles in
the same relationship. For instance, in:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: Store
          type: EntityType
        relationships:
          - name: ships_to_in_days
            roles:
              - player: Store
                name: destination
              - player: NrDays
            multiplicity: ManyToOne
            verbalizes: [ "{Store} ships to {Store:destination} in {NrDays}" ]
```

the role name `destination` distinguishes the second `Store`-playing role from the first in
this relationship.

Expressions that are used to define derived_by rules and requires constraints will refer to
roles by name -- the name defaulting to the concept that plays the role unless an explicit
role name is provided. In any expression that involving links of the `Store.ships_to_in_days`
relationship can then use the variables `Store` and `destination` to refer to objecs that
play these two `Store`-playing roles without ambiguity.

#### Multiplicities

If a relationship comprises more than one role, objects that play the last role could be functionally
dermined by a tuple of objects that play the other roles. This knowledge is declared using a `ManyToOne`
multiplicity constraint. In the examples above, the constraint declares that each person earns at most
one salary and that for each pair of stores, the former ships to the latter in at most one number of
days. For relationships of ternary and higher arity, the multiplicity applies to the n-th role, meaning
the object that plays the n-th role is functionally determined by the tuple of objects that play
the first n-1 roles.

In the special case of a binary relationship, one might declare a `OneToOne` multiplicity, which
indicates the relationship is many-to-one in both directions. For instance, the `Person.nr`
relationship is one-to-one because each person is assigned at most one social security number
and each social security number is assigned to at most one person.

In the absence of any multiplicity, we make no assumptions of functional dependencies among
any of the roles.

### Identifying relationships

Many conceptual models distinguish one or more relationships to use when referencing entity-type
objects in expressions and queries. The `Person.nr` relationship can be used to reference a
person by their social security number; while the pair of relationships `License.acct` and
`License.seat_nr` can be used to reference a license by its associated account and seat number.
These identifier relationships are always binary, and their first role is always played by the
concept the relationship is used to reference.

### Derivation expressions

Concepts and relationships may be derived using expressions. Think of a derived concept or 
relationship as a conceptual view whose objects or links are derived from those of other
concepts or relationships. For instance:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: Person
          type: EntityType
        relationships:
          - name: parent_of
            roles:
              - player: Person
                name: "child"
            verbalizes: [ "{Person} is a parent of {Person:child}", "{Person:child} is a child of {Person}" ]
          - name: ancestor_of
            roles:
              - player: Person
                name: "descendant"
            derived_by:
              - "Person.parent_of(descendant)"
                "Person.ancestor_of.parent_of(descendant)"  
          - name: taxed_at
            roles:
              - player: TaxRate
            derived_by:
              - "10.0 WHERE ( Person.files_single AND Person.earns <= 11925 )"
              - "10.0 WHERE ( Person.files_married_joint AND Person.earns <= 23850 )"
              - ...
```

declares two derived relationships -- `ancestor_of` and `taxed_at`. Each link of `Person.ancestor_of`
relates a person to one of its descendants. The two expressions form the base and recursive cases for
this calculation. In the base case, a `Person` as an ancestor of some `descendant` if that `Person`
is the parent of that descendant. And in the recursive case, a `Person` is an ancestor of some
`descendant` if that `Person` is an ancestor of the parent of that `descendant`. Notice in this
example how role names are used to disambiguate the two `Person` roles in this relationship.

Each link of `Person.taxed_at` links a `Person` object to a `TaxRate` that is derived using
expressions that determine the rate based on the person's filing status and how much they earn.
If, for some person, none of the expressions can be evaluated, then the relationship will have
no link involving that person.

Expressions that derive a relationship are interpreted as rules for constructing the links of the
relationship in the same way that a SQL query is interpreted as a rule for constructing the rows
of a new table. Each expression must therefore reference each role of the relationship, either
explicitly or implicitly. If an expression evaluates to some object (like 10.0 in the two examples
here) then that object will implicitly play the last role, and the expression must reference each
of the other roles explicitly. If an expression does not evaluate to any object, then it must
explicitly reference each role.

A derived concept is one whose population is derived from those of its supertype concepts
using one or more expressions. For instance:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: Employee
          type: EntityType
          extends: [Person]
          derived_by: [ "EXISTS ( Person.earns )" ]
```

declares that the population of Employee is derived from the population of Person by
classifying each Person who earns some salary as a Employee.

### Requires

The requires list contains expressions that give additional semantics to a concept or relationship
by declaring conditions that must hold over their populations. When applied to a concept, each
expression must reference the concept, as in:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: SocialSecurityNr
          type: ValueType
          extends: [Integer]
          requires: [ "0 < SocialSecurityNr", "SocialSecurityNr <= 999999999" ]
```

When applied to a relationship, each expression must reference one or more roles of the
relationship. For instance, in:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: Item
          type: EntityType
        relationships:
          - name: offers_in
            roles:
              - player: Store
            verbalizations: [ "{Item} is offered for sale in {Store}", "{Store} offers sale of {Item}" ]  
          - name: total_sales_in
            roles:
              - player: Store
              - player: Amount
            verbalizations: [ "{Item} sold for cumulative {Amount} in {Store}" ] 
            requires:
              - "Amount > 0.0"
              - "Item.offers_in(Store)"
```

the first expression requires any value that plays the `Amount` role to be positive while the second
requires any item that has sales in some store to be offered in that store.

## Ontology maps

Logical to conceptual schema mappings declare how to map field values to conceptual objects
and links. Just as ontologies are orgnaized into hierarchical components, so are ontology maps.
Each component map declares how to populate a concept with objects and any relationships that
are primarily keyed by that concept with links. These declarations are formed from patterns
of expressions that reference one or more fields in a logical model that is declared using
the OSI core semantic model spec.

Entity-type objects (entities) are opaque and must be mapped to via their identifying
relationships. These relationships are always binary, with the first role played by
the entity type itself. In the common case where an entity type is identified by a
single relationship whose second role is played by a value type, a value map suffices
to declare the existence of entities of that type or to look them up when mapping to
links of a relationship.

An entity map is an array of identifier maps:

| Field | Type | Required | Description |
|--------------|----------|-----|-------|
| `role`   | string   | Yes | Expression that indicates the role the mapped objects will play |
| `value`  | string   | if no `entity`  | Expression that maps to a value (when role played by a value type) |
| `entity` | list     | if no `value`  | Entity map that maps to an entity (when role played by an entity type) |

each of which is either a value map or a nested entity map that looks up an object to map to some
role of an identifier relationship.

Building on value and entity maps, we can declare precisely how fields determine the entities of
a concept and the links of every relationship in which that concept plays the first role. 

#### Entities (entity maps that determine the existence of objects of some entity type)

When a concept is an entity type, it can declare an `entities` array, each element of which
is an entity map.

For instance, in:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - name: PersonComponent
        concept:
          name: Person
          type: EntityType
          identify_by: [ nr ]
        relationships:
          - name: nr
            roles:
              - player: SocialSecurityNr
            multiplicity: OneToOne
            verbalizes: [ "{Person} is identified by {SocialSecurityNr}" ]
ontology_maps:
  - name: EnterpriseOntologyMap
    ontology: EnterpriseOntology
    logical_model:
      name: EnterpriseSemanticModel
      datasets:
        ...
    component_maps:
      - component: PersonComponentMap
        object_maps:
          - referent: Person
            identifier:
              - relationship: nr
                object:
                  referent: SocialSecurityNr
                  expression: PERSONS.SSN
    ...
```

uses a value map to declare how values from the `SSN` field of dataset `PERSONS` are used
to form `SocialSecurityNr` values that map to the second role of the `nr` relationship.
Because each link in that relationship associates a `SocialSecurityNr` value to some
unique `Person` entity, this value map suffices to associate each distinct `SSN` value
in the dataset to a distinct `Person` object in the ontology.

A more interesting example maps field values to `OrderLineItem` entities, which are
identified using two relationships:

```yaml
ontologies:
  - name: EnterpriseOntology
    components:
      - concept:
          name: OrderLineItem
          type: EntityType
          identify_by: [ "nr", "order" ]
          requires: [ "OrderLineItem.nr", "OrderLineItem.order" ]
        relationships:
          - name: nr
            roles: [ player: LineNr ]
            multiplicity: ManyToOne
          - name: order
            roles: [ player: CustOrder ]
            multiplicity: ManyToOne
ontology_maps:
  - ontology: EnterpriseOntology
    logical_model:
      name: EnterpriseSemanticModel
      datasets:
        ...
    component_maps:
      - component: OrderLineItemComponentMap
        object_maps:
          - referent: OrderLineItem
            identifier:
              - relationship: nr
                object:
                  referent: LineNr
                  expression: LINEITEMS.L_LINENUMBER
              - relationship: order
                object:
                  referent: Order
                  identifier:
                    relationship: CustOrder.nr
                    object:
                      referent: OrderNr
                      expression: LINEITEMS.L_ORDERKEY
```

This mapping contains a single entity map with two role mappings -- a value map that maps the
`L_LINENUMBER` field to the second role of the `nr` relationship, and a nested entity map that
maps `CustOrder` objects to the second role in the `order` relationship.

#### Links (mappings that determine the existence of relationship links)

Each relationships is grouped under the concept that plays its first role. A concept's
`links` array maps field schema to relationships. Each array element is a tree that
concisely declares how field values map to the links of one or more of these relationships.
More precisely, the path from the root of a tree to a node describes how to map to tuples
of objects that form the links of the relationship that is named by the node. These structures
leverage the hierarchical nature of YAML to avoid duplication in the typical case when the
fields of a single dataset map to many relationships.

Each tree node has the following schema:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `concept` | string | when level > 1 | Concept whose objects are looked up by this node |
| `value` | string | if no `entity` | Expression that computes a value (when concept is a value type) |
| `entity` | list | if no `value` | Entity map that looks up an entity (when concept is an entity type) |
| `relationship` | string | No | Relationship whose links are mapped to by the path to this node |
| `children` | list | No | List of child nodes in the tree |

Each node must provide either a `value` or an `entity` by which to look up objects but never both.
The level of each node coincides with the arity of the associated relationship. So a root node
could map to a unary relationship, a node at level 2 could map to a binary relationship, and so
forth.

For instance, the `links` array declared here:

```yaml
ontologies:
  - name: EnterproseOntology
    components:
      - concept:
          name: Item
          type: EntityType
          identify_by: [ nr ]
        relationships:
          - name: nr
            roles: [ concept: SkuNr ]
            multiplicity: OneToOne
            verbalises: "{Item} is identified by {SkuNr}"
          - name: active     # A unary relationship
            verbalizes: [ "{Item} is actively sold" ]
          - name: active_in
            roles: [ concept: Store ]
            verbalises: [ "{Item} is actively sold in {Store}" ]
          - name: returned_in_for
            roles: [ concept: Store, concept: Amount ]
            verbalizes: [ "{Item} returned in {Store} for {Amount}" ]
            multiplicitly: ManyToOne
          - name: sold_in_for
            roles: [ concept: Store, concept: Amount ]
            verbalizes: [ "{Item} sells in {Store} for {Amount}" ]
            multiplicitly: ManyToOne
ontology_maps:
  - name: flights_map
    description: Example mapping of logical fields to ontology concepts and relationships
    ontology: flights
    logical_model:
      name: Logical_Flights_Model
      description: Logical model for flight data
      datasets:
        ...
    component_maps:
      - component: ItemComponentMap
        object_maps:
          - referent: Item
            identifier:
              relationship: Item.nr
              object:
                referent: ItemNr
                expression: ITEMS.SKU
        link_maps:
          - maps:
              referent: Item
              identifier:
                relationship: Item.nr
                object:
                  referent: ItemNr
                  expression: METRICS.SKU
            populates: Item.active
            children:
              - maps:
                  referent: Store
                  identifier:
                    relationship: Store.nr
                    object:
                      referent: StoreNr
                      expression: METRICS.STORE
                populates: Item.active_in
                children:
                  - maps:
                      referent: Amount
                      expression: METRICS.SALES
                    populates: Item.sold_in_for
                  - maps:
                      referent: Amount
                      expression: METRICS.RETURNS
                    populates: Item.returned_in_for
```

describes a tree with one root node, one node at level 2, and two nodes at level 3.
Each node maps fields of the `METRICS` dataset to links of four different relationships,
and notice how the mapping to `Item` objects is declared once even though `Item` plays a
role in all four of the relationships and that the mapping to `Store` objects is declared
once even though `Store` plays a role in three of the relationships.


---

## Complete Example

Here's a complete ontology example showing all components working together:

```yaml
```

## Version History

- **0.1.1** (2026-05-20): Support for both logical and conceptual modeling layers (ontologies)
  - Core ontology structure
  - Support for concepts, relationships, and logical -> conceptual schema mappings
  - Renamed relationships in the logical layer to join paths to avoid conflict with
    relationships in the conceptual layer

---

## License

See LICENSE file for details.
