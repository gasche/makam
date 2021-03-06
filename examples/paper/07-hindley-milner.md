(Text is very much WIP.)

<!--
```makam
%use "05-type-synonyms.md".
%testsuite literate_tests.
```
-->

Let's now do Hindley-Milner let-polymorphism:

```makam
let : term -> (term -> term) -> term.
```

Easy so far.

The inference rule looks like this:
\begin{displaymath}
\inferrule{
  \Gamma \vdash e : \tau \\
  \vec{a} = \text{fv}(\tau) - \text{fv}(\Gamma) \\
  \Gamma, x : \forall \vec{a}.\tau \vdash e' : \tau'
}{
  \Gamma \vdash \text{let} \; x = e \; \text{in} \; e' : \tau'
}
\end{displaymath}

(We have not added any side-effectful operations, so no need to add a value restriction.)

This is easy to transcribe in Makam, assuming a predicate for generalizing a type:

```makam
generalize : typ -> typ -> prop.

typeof (let E F) T' :-
  typeof E T,
  generalize T Tgen,
  (x:term -> typeof x Tgen -> typeof (F x) T').
```

So we need to do the following:

- something that picks out free-variables from a term -- or, in our setting, uninstantiated meta-variables
- something that picks out free-variables from the local context
- a way to turn something that includes meta-variables into a `forall` type

This predicate picks out the first metavariable of a certain type it finds. It uses `generic.fold`
which is another generic operation, defined similarly to `structural_recursion`, but which performs
a fold over arbitrary types.

```makam
findunif : [A B] option B -> A -> option B -> prop.
findunif (some X) _ (some X).
findunif none (X : A) (some (X : A)) :- refl.isunif X.
findunif In X Out :- generic.fold findunif In X Out.

findunif : [A B] A -> B -> prop.
findunif T X :- findunif none T (some X).
```

Note that the second rule, the important one, will only match when we encounter a metavariable
of the same type as the one we require, as we do type specialization.

Now let's add something, that given a specific meta-variable and a specific term, replaces the
meta-variable with the term. We will see later why this is necessary. Here we will need another
reflective predicate, `refl.sameunif` that succeeds when its two arguments are the same exact
metavariable. As opposed to unifying two metavariables, this allows us to "pick out" occurrences
of a specific metavariable.

```makam
replaceunif : [A B] A -> A -> B -> B -> prop.
replaceunif Which ToWhat Where Result :-
  refl.isunif Where,
  if (refl.sameunif Which Where)
  then (eq (dyn Result) (dyn ToWhat))
  else (eq Result Where).
replaceunif Which ToWhat Where Result :-
  not(refl.isunif Where),
  structural_recursion (replaceunif Which ToWhat) Where Result.
```

A last auxiliary predicate will allow us to check whether a specific metavariable exists
within a term:

```makam
hasunif : [A B] B -> bool -> A -> bool -> prop.
hasunif _ true _ true.
hasunif X false Y true :- refl.sameunif X Y.
hasunif X In Y Out :- generic.fold (hasunif X) In Y Out.

hasunif : [A B] A -> B -> prop.
hasunif Term Var :- hasunif Var false Term true.
```

We are now ready to implement `generalize`. Base case: there exist no unification variables
within a type:
```makam
generalize T T :- 
  not(findunif T X).
```

Recursive case: there exists at least one unification variable. We will pick out that unification
variable, abstract over it and repeat the process to pick out any remaining ones.  We will check
whether we are allowed to generalize by getting something that holds all `typ`s in the current
variable environment -- that is, all `T`s for any `typeof x T` local assumptions -- and making sure
that the current unification variable does not occur in that.  Getting the types in the environment
is done through the `get_types_in_environment` predicate, and we will leave the type of its result
abstract for the time being.

```makam
get_types_in_environment : [A] A -> prop.

generalize T Res :-
  findunif T X,
  (x:typ -> (replaceunif X x T (T' x), generalize (T' x) (T'' x))),
  get_types_in_environment Types,
  if (hasunif Types X)
  then (eq Res (T'' X))
  else (eq Res (forall T'')).
```

What can `get_types_in_environment` be? We could change all our typing rules to add a list argument
that holds all the types that we put in the context, and thread it through all our predicates.
However, again using reflective predicates, there is an easier way to do that: we can simply get
all the local assumptions for the `typeof` predicate for terms, which will exactly correspond
to the local assumptions for the current set of free variables:

```makam
get_types_in_environment Assumptions :-
  refl.assume_get (typeof : term -> typ -> prop) Assumptions.
```

We're done!

Example, easy:

```makam
typeof (let (lam _ (fun x => x)) (fun id => id)) T ?
>> Yes:
>> T := forall (fun a => arrow a a).
```

Another example, where the problem of naive generalization shows up:

```makam
typeof (let (lam _ (fun x => let x (fun y => y)))
            (fun z => z)) T ?
>> Yes:
>> T := forall (fun a => arrow a a).
```

(Just checking the issue where we don't remove all unification variables in the context -- this
is a hack, if we need to do this we can show the above in two steps instead:)

```makam
(get_types_in_environment [] ->
  typeof (let (lam _ (fun x => let x (fun y => y)))
            (fun z => z)) T) ?
>> Yes:
>> T := forall (fun a => arrow a (forall (fun b => b))).
```
