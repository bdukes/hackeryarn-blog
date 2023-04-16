---
title: When DRY goes wrong
date: 2023-02-20
published: false
---

DRY has become a mantra throughout the industry. Any time that some developers see repetitive code, they want to extract it into its own function. It has become almost robotic to extract code and few people give a second thought to whether it makes sense all the time.

Unfortunately, poor application of DRY can lead to brittle code where it's scary to change functionality because it could have a huge ripple effect on the rest of the code base. In this article I hope to show how this happens and cover other ways to look for abstractions. And even make an argument that duplication of good is good in the right setting.

## An example

First off, let's ignore other forms of abstractions like classes, typeclasses, traits, or interfaces. I know how tempting it will be to use a cleaner form of abstraction throughout this article, but doing that would send us down a whole new set of rabbit holes.

We will stick to using plain functions so that things are easy to reason about and apply more broadly. After all, functions are the common denominator among just about every programming language out there. 

Our example will consist of an application to calculate salaries of employees at different levels. We start out with two types of employees. And even with this little code, we already notice repetition creeping up:

```python
def calculate_ic_compensation(employee):
    salary = 0
    # some calculations before...
    if employee.bonus:
      salary += employee.bonus / 12
    # more calculations after...

def calculate_manager_compensation(employee):
    salary = 0
    # some calculations before...
    if employee.bonus:
      salary += employee.bonus / 12
    # more calculations after...
```

Now, would you want to extract this code into its own function? I know I would. And is the repetition a good enough reason for the extraction?

I think that this is absolutely the right case for extraction, but not because of the repetition. There is a more subtle reason why we should extract this code, that we will get to later.

For now, let's proceed with extracting the repetitive code:

```python
def calculate_bonus(employee):
    if employee.bonus:
        return employee.bonus / 12
    return 0

def calculate_ic_compensation(employee):
    salary = 0
    # some calculations before...
    salary += calculate_bonus(employee)
    # more calculations after...

def calculate_manager_compensation(employee):
    salary = 0
    # some calculations before...
    salary += calculate_bonus(employee)
    # more calculations after...
```

When we extract this code, we create a clean separation of concerns. We have a single place that deals with bonus calculations and helper functions that use the bonus calculation in their calculation for employee's compensation.

## How problems arise

As our code lives for a few weeks or months, and as code does, it grows. We have a few more functions using `calculate_bonus`. Calculations for directors and VPs also started using this same function. This company is all about equality, so everyone so far has the same bonus structure.

But we just got a requirement to handle C-Level employee compensations, and their bonus structure is a little different. For every year that they were with the company, they get an additional $2,000 bonus.

How would you handle this situation?

We have two options. We can add this logic into the new `calculate_clevel_compensation`, or we can expand `calculate_bonus`.

Adding this logic into `calculate_clevel_compensation` doesn't quite feel right. We've isolated our concept of bonus calculation into its own nice little function, so it only make sense to keep all the bonus calculating logic together. Here is how we might expand `calculate_bonus` to handle the new requirement:

```python
def calculate_bonus(employee, clevel=False):
    bonus = 0 

    if employee.bonus:
        bonus += employee.bonus / 12

    if clevel:
        bonus += 2000 * employee.years_worked

    return bonus
```

On the surface this looks like a good solution. By providing a default value for `clevel` we avoid having to update the old function invocations, and our new function just has to provide one flag.

So what would happen if another position gets a different bonus structure? We could just add another flag, add another `if`, and make sure all the right places use the right flags.

And what if there is some shared logic between two of those flags? Or what if multiple flags signal some kind of other special condition? You can see how this innocent function can quickly get out of hand.

This solution has two serious drawbacks.

First off, it introduces a new options named `clevel` which has nothing to do with the concept of a bonus calculation. It's related to the role which should be fully encapsulated by the function that calls `calculate_bonus`.

This bleeding over of concepts makes code significantly harder to follow. Because `clevel` tells us nothing about how it might impact the bonus calculation, we have to read two functions and keep track of the related parts in our minds. And if we have more of these types of "helper" functions, we end up having to keep a lot of information in our head while trying to understand a single function.

The other problem is closely related to the first, but is more insidious. Having the level as part of the bonus calculation tightly couples this function to working with only functions that deal with levels. We simply can't re-use this function in a context that does not know about employee levels.

So if we want to run a simulation that shows the impacts of different bonus structures, we have to introduce levels into this simulations even though we don't care about levels when simulating optimal bonus structures. This really kills DRY's promise of re-usability.

These are some serious issue, but not all is lost. Let's look at a way that we can identify and fix these types of problems.

## Abstraction Barriers

A better approach of extracting pieces of code comes form a concept significantly older than DRY. Abstraction barriers were first formally introduced in [Structure and Interpretation of Computer Programs](https://sarabander.github.io/sicp/html/2_002e1.xhtml#g_t2_002e1_002e2), published in 1984.

I encourage you to read the section linked above, if you're not afraid of too many parentheses. But for the purpose of this article I can sum up this as: build abstractions in well defined layers.

Without reading the linked section of SICP, you might wonder what constitutes a layer. The most basic layer that every program has is the language itself. All the built in functions and operators create an abstraction layer over the machine code that executes the program. We would almost never want to go down below this level, although there are some times when we must.

The next most common layer is made up by the libraries that we import. They usually provide further abstraction over the language allowing us to describe our intent in terms that belong to the domain of the library. The layers in our applications should work in almost the same way.

When we started our program, we were working in a layer that deals with employees and their levels. By extracting our function, we unintentionally introduced a new layer. The unintentional extraction made this layer look like it was for calculating bonuses, but that's too narrow. We created a layer for working with financial calculations. 

If you prefer to think of it in terms of domains: The first layer belongs to the HR domain. It deals with levels, and what those levels entail. The second layer belongs to the accounting domain. It deals with the details that go into calculating certain financial concepts -- bonuses being one of them. If we need clarification on terminology or functionality, this distinction points us towards the right group to talk to.

## Apply Abstraction Barriers

After identifying these layers, it's clear that we need to have separate terminology at each layer that doesn't allow them to bleed together. And since the programs we write model the real world, we can borrow this terminology from the domain experts:

```python
def calculate_bonus(employee, yearly_increase=0):
    bonus = 0 

    if employee.bonus:
        bonus += employee.bonus / 12

    if clevel:
        bonus += yearly_increase * employee.years_worked

    return bonus
```

We did two things here. We renamed the variable to something that makes more sense in the concept of calculations, and we went from a boolean to passing an integer.

Using a boolean was permissible when the layers were intertwined. But once we separated them, it becomes clear that the amount of the yearly increase has nothing to do with the calculation. Our calculation only needs to know how to use this amount and the amount itself needs to come from the higher level.

This demonstrates the key concept of abstraction barriers that we haven't explicitly mentioned, yet. Each layer can only use the layers below it. This practice keeps the code looking like a neat tree of dependencies instead of a tangled mess. The term spaghetti code refers to the mess we would have if we didn't follow this principle. Calling across layers or up to higher level quickly leads to circular dependencies and concepts becoming tangled with unrelated once (just like we saw when calculations were put int he same layer as the levels logic).

## Taking it further

You might have noticed the comments in the original example that stated that each role had more calculations before and after. But our bonus calculation was the first introduction to the calculations layer. So what should we do with all these calculations.

Well, we might already have helpers for some of these. In that case, we should review these helper function for any concept sharing and pull them into our calculations domain.

Other parts, however, might be entangled with level logic. We should carefully review these and make the relevant parts stand alone calculations. Yes, even if we are not re-using them we should put them in the appropriate abstraction layer.

Let's look at what a final function might look like before we talk about the advantages of having everything related to calculations in its own layer.

```python
def calculate_clevel_compensation(employee):
    salary = 0
    salary += calculate_monthly_salary(employee)
    salary += calculate_bonus(employee, yearly_increase=2000)
    salary += calculate_overtime(employee)
    return salary
```

You can see that we no longer need to comb through complex logic. This function simply calls to the calculation layer and combines all the returned values with a `+`. That's all that our level calculations should really do. With this layout we can even move things around, insert steps, or remove them with full confidence. 

## Small functions over options

The above solution is much clearer, but it's still not perfect. Any time a function takes an optional argument, it should strike us as a red flag. And this might be the most controversial statement int he article. After all, many standard libraries have functions with tons of flags. Python's `print` takes 4 optional arguments after all. But I would argue that it would be better to have multiple functions for the different scenarios instead of having all those options.

If we have multiple functions, we unlock far more flexibility. Continuing with our example, we get a requirement that under a specific scenario we actually need to look at the full bonus amount before determining the yearly increase. If we try to keep the calculation inside `calculate_bonus`, we would end up adding another flag that while relevant, is starting to morph the function into holding a lot of complexity.

Multiple flags also hide some of the interactions that they might have with each other. At a certain point questions arise and we are back and needing to read the function with tons of flags before we can understand what it's doing.

So the solution is to just introduce another function. So our final version of `calculate_clevel_compnsation` looks like:

```python
def calculate_clevel_compensation(employee):
    salary = 0
    salary += calculate_monthly_salary(employee)
    salary += calculate_bonus(employee)
    salary += calculate_yearly_increase_bonus(employee, 2000)
    salary += calculate_overtime(employee)
    return salary
```

## Are we just moving repetition around?

At this point, you might read all our `calculate_x_compensations` and think that we could dry those up too. After all some steps repeat in every function. But you must resist this urge.

Repetition actually has a use. It gives us flexibility. By having these higher level concepts repeated, we leave the door open to easily modify the steps for each one. This might sound like I am advocating for huge functions, and at a certain level of abstraction I am. We have to have repetition somewhere, and the abstractions barriers that we built earlier give us an easy way to see where it belongs.

The top layer of abstraction is the layer closest to the business domain. This is the layer that is most likely to change frequently, and since repetition gives us flexibility, this is the level that needs the most of it. As we move down our stack of layers, they should become terser and care more about fully encapsulating concepts in nice abstractions.

We should also pay close attention to how often certain levels change as the code base evolves. If we end up having a low level of abstraction change often, we should modify it so that the parts that change often happen at our highest level. If we continue modifying the low level often, we cause huge ripple effects throughout our code base. And the lower the level, the more code depends on it. This is why major version language changes are so painful. They ripple in ways we can't even track.

So to thumb this up. I want my lowest level to have functions with just a few lines so that they rarely change. And at my highest level, I am OK with 100 line long functions that handles a complex process. But only if 99% of that functions is built on abstractions that I have created in lower levels. After all if a process is complex we don't want to make it more complex by having to jump around 10 "helper" functions to figure out what's actually going on.

## In conclusion

The next time you see a concept repeating, try to go beyond just extracting common functionality. Take the time to put it into the appropriate module, rename any options to concepts that belong at the correct abstraction layer, and make your future self grateful to the you of today.
