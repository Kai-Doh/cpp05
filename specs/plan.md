# CPP05 — Implementation Plan (Repetition and Exceptions)

**Source**: `SUBJECT.md` | **Standard**: C++98 | **Build order**: ex00 → ex01 → ex02 → ex03 (strictly sequential — each exercise extends/refactors the previous one)

## 1. Global constraints (apply to every exercise)

- **Compiler/flags**: `c++ -Wall -Wextra -Werror`; code must also compile with `-std=c++98` added.
- **Forbidden**: `using namespace`, `friend`, STL containers (`vector`, `map`, `list`, ...), `<algorithm>`, `*printf()`, `*alloc()`/`free()`. `std::string`, `std::exception`, `<iostream>`, `<sstream>`, `<fstream>`, `rand()`/`srand()` are all fine.
- **Orthodox Canonical Form (OCF)** required for every class **except exception classes** (explicitly exempted in ex00). OCF = default constructor, copy constructor, copy assignment operator, destructor — all `public`. Additional (parameterized) constructors are allowed on top of these four.
  - Where a class has `const` data members, the copy-assignment operator cannot reassign them. The accepted pattern: copy only the non-const members; leave `const` members untouched (they were fixed at construction).
  - The default constructor for classes with `const` members must still compile — give the `const` members a sensible default (e.g. empty name, lowest valid grade `150`, `false` for booleans) so the default-constructed object is in a valid, non-throwing state.
- **No implementations in headers** (except templates — none needed here).
- **Include guards** on every header; each header must compile standalone (include everything it uses).
- **No memory leaks**: every `new` must be matched by a `delete` (handled by the *caller* of `Intern::makeForm` in ex03 — document this in comments/README, not code comments).
- **Exception specs**: every custom exception derives (directly or indirectly) from `std::exception`, overrides
  `virtual const char* what() const throw();` (note: C++98 spec is `throw()`, not `noexcept`). All must be catchable as `std::exception&`.
- **Int → string conversion**: no `std::to_string` (C++11). Use `<sstream>` (`std::ostringstream`) wherever a grade/number must be embedded in a message or stream.
- **Naming**: UpperCamelCase class names; file names match class names (`ClassName.hpp`, `ClassName.cpp`).

## 2. Exception hierarchy (cumulative across exercises)

```
std::exception
├── Bureaucrat::GradeTooHighException   (ex00)
├── Bureaucrat::GradeTooLowException    (ex00)
├── AForm::GradeTooHighException        (ex01 as Form::..., renamed in ex02)
├── AForm::GradeTooLowException         (ex01 as Form::..., renamed in ex02)
└── AForm::FormNotSignedException       (new in ex02, thrown by execute())
```

Each is a small nested class with no members, only `what() const throw()` returning a static C-string literal (e.g. `"grade too high"`). They are exempt from OCF.

## 3. Exercise 00 — `Bureaucrat`

**Dir**: `ex00/` | **Files**: `Makefile`, `main.cpp`, `Bureaucrat.hpp`, `Bureaucrat.cpp`

### Class: `Bureaucrat`

| Member | Type | Notes |
|---|---|---|
| `_name` | `const std::string` | set once at construction |
| `_grade` | `int` | 1 (highest) .. 150 (lowest) |

Constants `MIN_GRADE = 1`, `MAX_GRADE = 150` — declare as `static const int` in the header (in-class initialization of integral `static const` is legal in C++98).

### Constructors / OCF

1. `Bureaucrat()` — default ctor: `_name("")`, `_grade(MAX_GRADE)` (always valid, never throws).
2. `Bureaucrat(const std::string& name, int grade)` — validates `grade` via a private static helper `checkGrade(int)`; throws `GradeTooHighException` if `grade < MIN_GRADE`, `GradeTooLowException` if `grade > MAX_GRADE`.
3. `Bureaucrat(const Bureaucrat& other)` — copy ctor, copies `_grade`; `_name` initialized from `other._name` (allowed in ctor init list even though const).
4. `Bureaucrat& operator=(const Bureaucrat& other)` — copies only `_grade` (cannot touch `_name`, it's const). Return `*this`.
5. `~Bureaucrat()` — empty, nothing dynamically allocated.

### Member functions

- `const std::string& getName() const`
- `int getGrade() const`
- `void incrementGrade()` — `_grade--` (increment grade = numerically *decrease* value); if result `< MIN_GRADE` (i.e. `_grade == 1` already) throw `GradeTooHighException`.
- `void decrementGrade()` — `_grade++`; if result `> MAX_GRADE` throw `GradeTooLowException`.
- `private static void checkGrade(int grade)` — throws the appropriate exception if out of `[MIN_GRADE, MAX_GRADE]`, reused by ctor + increment/decrement.

### Exceptions (nested, exempt from OCF)

- `class GradeTooHighException : public std::exception { const char* what() const throw(); }`
- `class GradeTooLowException : public std::exception { const char* what() const throw(); }`

### Free function

- `std::ostream& operator<<(std::ostream& os, const Bureaucrat& b)` — non-member (friend is forbidden, so it uses only `getName()`/`getGrade()`), format: `"<name>, bureaucrat grade <grade>."` followed by newline at the call site/main as needed.

### `main.cpp` tests

- Construct with valid grade (e.g. 75), print it.
- Construct with `grade = 0` and `grade = 151` → catch `GradeTooHighException` / `GradeTooLowException` via `std::exception&`, print `e.what()`.
- Increment a grade-1 bureaucrat → catch `GradeTooHighException`.
- Decrement a grade-150 bureaucrat → catch `GradeTooLowException`.
- Test copy ctor / assignment.

## 4. Exercise 01 — `Form`

**Dir**: `ex01/` | **Files**: ex00 files + `Form.hpp`, `Form.cpp`

### Class: `Form`

| Member | Type | Notes |
|---|---|---|
| `_name` | `const std::string` | |
| `_isSigned` | `bool` | starts `false` |
| `_gradeToSign` | `const int` | 1..150 |
| `_gradeToExecute` | `const int` | 1..150 |

All **private** (subject explicitly says private, not protected — important because ex02 will need to revisit this when introducing the abstract base).

### Constructors / OCF

1. `Form()` — default ctor: `_name("")`, `_isSigned(false)`, `_gradeToSign(MAX_GRADE)`, `_gradeToExecute(MAX_GRADE)`.
2. `Form(const std::string& name, int gradeToSign, int gradeToExecute)` — validates both grades with the same `checkGrade` pattern as `Bureaucrat`, throwing `Form::GradeTooHighException` / `Form::GradeTooLowException`.
3. Copy ctor — copies `_isSigned`; const members copied via init list from `other`.
4. Copy assignment — copies only `_isSigned` (the three `const` members can't be reassigned).
5. Destructor — empty.

### Member functions

- `const std::string& getName() const`
- `bool getIsSigned() const`
- `int getGradeToSign() const`
- `int getGradeToExecute() const`
- `void beSigned(const Bureaucrat& b)` — if `b.getGrade() <= _gradeToSign`, set `_isSigned = true`; otherwise throw `Form::GradeTooLowException`.
- `private static void checkGrade(int grade)` — same role as in `Bureaucrat`, but throwing `Form::GradeTooHighException` / `Form::GradeTooLowException`.

### Exceptions (nested, exempt from OCF)

- `Form::GradeTooHighException`
- `Form::GradeTooLowException`

### Free function

- `std::ostream& operator<<(std::ostream& os, const Form& f)` — prints name, signed status, grade to sign, grade to execute.

### `Bureaucrat` additions

- `void signForm(Form& form)` — note: **non-const** `Form&` because `beSigned` mutates it. Wrap `form.beSigned(*this)` in try/catch:
  - success → print `"<bureaucrat-name> signed <form-name>"`
  - failure → catch `std::exception&`, print `"<bureaucrat-name> couldn't sign <form-name> because <e.what()>"`

### `main.cpp` tests

- Build forms with valid/out-of-range grades (catch ctor exceptions).
- Sign a form with a bureaucrat whose grade is sufficient/insufficient (boundary cases: `grade == gradeToSign` must succeed).
- Print form before/after signing via `operator<<`.
- `signForm` success and failure message formats.

## 5. Exercise 02 — `AForm` + concrete forms

**Dir**: `ex02/` | **Files**: `Makefile`, `main.cpp`, `Bureaucrat.{hpp,cpp}`, `AForm.{hpp,cpp}`, `ShrubberyCreationForm.{hpp,cpp}`, `RobotomyRequestForm.{hpp,cpp}`, `PresidentialPardonForm.{hpp,cpp}`

This exercise **renames `Form` → `AForm`** and makes it abstract. `Bureaucrat` is copied over unchanged (signForm now takes `AForm&`).

### Class: `AForm` (abstract base)

Same four private members as `Form` (`_name`, `_isSigned`, `_gradeToSign`, `_gradeToExecute`) — still **private**, still belong to the base class as the subject requires.

#### OCF

Same 5 special members as `Form` in §4, but the constructor signature gains nothing new — concrete subclasses call it via their init list with hard-coded grade values.

#### Member functions

- Same getters as `Form`.
- `void beSigned(const Bureaucrat& b)` — unchanged logic.
- `void execute(const Bureaucrat& executor) const` — **non-virtual, concrete, implemented in `AForm`**. It performs the shared validation:
  1. if `!_isSigned` → throw `AForm::FormNotSignedException`
  2. if `executor.getGrade() > _gradeToExecute` → throw `AForm::GradeTooLowException`
  3. otherwise call the pure-virtual `executeAction()`
- `protected: virtual void executeAction() const = 0;` — **this is the pure virtual that makes `AForm` abstract**. Each concrete class implements the actual side-effect (write file / drill noises / pardon message). Keeping it `protected` and pure-virtual satisfies "execute(...) const member function added to the base form" while letting subclasses supply only their specific behavior — avoids duplicating the signed/grade checks in every subclass (the "more elegant" option the subject hints at).
- `private static void checkGrade(int grade)` — same pattern, throws `AForm::GradeTooHighException` / `AForm::GradeTooLowException`.

#### Exceptions (nested, exempt from OCF)

- `AForm::GradeTooHighException`
- `AForm::GradeTooLowException`
- `AForm::FormNotSignedException` — new, thrown by `execute()` when `_isSigned == false`.

#### Free function

- `std::ostream& operator<<(std::ostream& os, const AForm& f)` — same as `Form`'s.

### Concrete class: `ShrubberyCreationForm`

- Grades: sign `145`, exec `137`.
- Extra private member: `const std::string _target`.
- OCF: default ctor (`_target("")`, fixed grades 145/137, name `"ShrubberyCreationForm"`), parameterized ctor `ShrubberyCreationForm(const std::string& target)` calling `AForm("ShrubberyCreationForm", 145, 137)` + initializing `_target`, copy ctor/assign (assign copies inherited `_isSigned` only — `_target` is const), destructor.
- `protected: void executeAction() const;` (override) — opens `std::ofstream` on `(_target + "_shrubbery")`, writes ASCII-art trees, closes file. Use `<fstream>`, not C file functions.

### Concrete class: `RobotomyRequestForm`

- Grades: sign `72`, exec `45`.
- Extra private member: `const std::string _target`.
- OCF mirrors `ShrubberyCreationForm`.
- `executeAction() const` (override) — print drilling-noise message, then `rand() % 2` (seed with `srand(time(NULL))` once, e.g. in `main` or a static init) to decide between `"<target> has been robotomized successfully"` and a failure message. `<cstdlib>` + `<ctime>` are allowed (not in the forbidden list).

### Concrete class: `PresidentialPardonForm`

- Grades: sign `25`, exec `5`.
- Extra private member: `const std::string _target`.
- OCF mirrors the others.
- `executeAction() const` (override) — print `"<target> has been pardoned by Zaphod Beeblebrox"`.

### `Bureaucrat` additions

- `void executeForm(const AForm& form) const` — try `form.execute(*this)`:
  - success → print `"<bureaucrat-name> executed <form-name>"`
  - failure → catch `std::exception&`, print explicit error (`"<bureaucrat-name> couldn't execute <form-name> because <e.what()>"`)

### `main.cpp` tests

- For each concrete form: construct, attempt `execute` before signing → `FormNotSignedException`.
- Sign with low-grade bureaucrat → `AForm::GradeTooLowException` from `beSigned`.
- Sign with sufficient grade, then `execute` with insufficient-grade bureaucrat → `AForm::GradeTooLowException` from `execute`.
- Sign + execute with sufficient grade for all three forms:
  - verify `<target>_shrubbery` file is created with content.
  - run `RobotomyRequestForm` execute multiple times to observe both success/failure branches.
  - verify pardon message.
- All via `Bureaucrat::executeForm`, observing the two message formats.

## 6. Exercise 03 — `Intern`

**Dir**: `ex03/` | **Files**: ex02 files + `Intern.hpp`, `Intern.cpp`

### Class: `Intern`

- No data members required ("no name, no grade, no unique characteristics").
- OCF: default ctor, copy ctor, copy assignment, destructor — all trivial/empty since there's no state. (Default ctor here is genuinely meaningful, unlike the other classes.)

### Member function

- `AForm* makeForm(const std::string& formName, const std::string& target) const`
  - Returns a heap-allocated `AForm*` of the concrete type matching `formName` (`"shrubbery creation"`, `"robotomy request"`, `"presidential pardon"`), constructed with `target`.
  - Prints `"Intern creates <form-name>"` on success.
  - On unknown `formName`, prints an explicit error and returns `NULL` — **no exception needed**, this is a user-input error, not a program-state error.
  - **Avoid if/else-if chains**: build a small fixed-size lookup table (C-array, size 3) of `{ const char* name; AForm* (*create)(const std::string&); }`, where each `create` is a `private static` factory method (`createShrubbery`, `createRobotomy`, `createPresidential`) returning `new ShrubberyCreationForm(target)` etc. Loop over the array comparing `formName` with `std::string::operator==`, and call the matching function pointer. (A C-array of fixed size + function pointers is not a "container" in the forbidden STL sense.)

### `main.cpp` tests

- `Intern` creates each of the three valid form types via `makeForm`, prints confirmation.
- `makeForm` with an invalid name → prints error, returns `NULL` (check for `NULL` before use).
- For at least one created form: sign it with an adequately-graded `Bureaucrat`, then `executeForm` it — confirms the returned pointer is a fully functional `AForm*` (polymorphism via the base-class interface).
- `delete` every `AForm*` returned by `makeForm` (no leaks).

## 7. Makefile (per exercise, identical skeleton)

```makefile
NAME      = <exercise_binary_name>
CXX       = c++
CXXFLAGS  = -Wall -Wextra -Werror -std=c++98
SRCS      = main.cpp <ClassName.cpp ...>
OBJS      = $(SRCS:.cpp=.o)

all: $(NAME)

$(NAME): $(OBJS)
	$(CXX) $(CXXFLAGS) $(OBJS) -o $(NAME)

clean:
	rm -f $(OBJS)

fclean: clean
	rm -f $(NAME)

re: fclean all

.PHONY: all clean fclean re
```

ex02/ex03 `SRCS` accumulate all class `.cpp` files from earlier exercises plus the new ones, per the "Files to Submit" lists in `SUBJECT.md`.

## 8. Build order rationale

1. **ex00** establishes the OCF + exception + `checkGrade`/`operator<<` patterns reused everywhere.
2. **ex01** reuses that pattern for `Form` and introduces cross-class interaction (`Bureaucrat::signForm` calling `Form::beSigned`) — the template for ex02's `executeForm`/`execute`.
3. **ex02** is a refactor of ex01's `Form` into an abstract `AForm` plus three subclasses — must come after ex01 because the validation logic (`checkGrade`, `beSigned`) and the `operator<<`/getter set carry over almost unchanged.
4. **ex03** depends on the full `AForm` hierarchy from ex02 to implement the factory in `Intern::makeForm`.
