# Terraform Fundamentals (Variables, Locals, data types, additional functions): 12/30/25

Notes for covering Variables and associated concepts of parameterization; locals and associated concepts of readability and DRY principles; data types and conversions; and related functions. This class is critical preparation for modules, working in multiple enviorments, improving code readability and so on. 

## Repo Layout
- [Syntax Demo](./syntax-demo/): This contains no providers and serves as a starting basis for exploring these features in terraform. Of course, no AWS resources will be deployed with this. 
- [AWS Infrastructure code](./simple-code-no-refactor/): We will use this code as a starting block to refactor later on. 
- [Refactored Code](./simple/): This code is my personal refactored code and one of the goals for class. Please note this code may not follow some best practices for variables (such as overuse) as utilization was prioritized. 
- [For later](./advanced/): This is the [refactored code](./simple/) with additional terraform features added that will not be covered tonight and is for my notes. No gurantees it works properly. 

```bash
# navigate into your project folder 
# for example, if you use theowaf folder, do something like this:
# cd ~/Documents/TheoWAF/class7/AWS/Terraform
# no need to create a folder of today's date

git clone --no-checkout https://github.com/aaron-dm-mcdonald/Class7-notes.git
cd Class7-notes
git sparse-checkout set --cone 123025/syntax-demo 123025/simple-code-no-refactor
git checkout
cd .. && mv ./Class7-notes/123025/ ./variable-lab && rm -rf ./Class7-notes/
```

---

## Table of Contents

1. What Terraform evaluates first 
2. Variables vs locals (what & when)
3. Variable types (string, number, bool, list, map, object)
4. Defining and setting variable values
5. Outputs and accessing data
6. Conditionals (ternary operator)
7. Splat operator
8. Arithmetic expressions
9. Common functions
   - Map functions
   - List & set functions
   - Boolean & numeric functions
10. Type conversions
11. Variable validations
12. Common beginner errors
13. Suggested teaching file layout

---

## 1. Terraform’s evaluation model (important context)

Terraform evaluates **expressions**. Everything in terraform that is provided to an argument is an **expression**. 

Expressions are the following:
- literals ("Hello", 20, etc)
- dot references (aws_instance.example.subnet_id for example)
- functions (such as the file() function)
- arithmetic (20 + 5)

Essentially this means in basically all situations, the right side of the equals sign is an **expression** and during the execution plan generation or provisioning of infratructure they are evalulated to a **value** (which has a data type that will be discussed later). This is a critical concept to know. 

Finally, understand that while everything is an **expression**, not all **expressions** are allowed in certain places. Sometimes functions are not permitted or dot references are not permitted as an example (we will discuss one example later on).

Docs:  
https://developer.hashicorp.com/terraform/language/expressions

---

## 2. Variables vs locals

Before we continue we must clarify two things for (likely) two sets of students:

1) To students not familar with CS terminology or who have not used a scripting/programming language before
  - Parameters: Simplied as much as possible in this context, a parameter is something that can be changed depending on the situation. 
  - We use parameterization to try to prevent people that are not DevOps professionals from modifying our Terraform code or at least streamlining this process. 
  - This prevents minor errors, speeds deployments, and allows different teams to deploy infrastructure (somewhat) more independetly while encouraging them to avoid introducing state drift via ClickOps.

2) Students familar with the ideas of function arguments and variables in other languages: 
  - Please forgive Hashicorp for the poor naming conventions.
  - Variables are used essentially like argument parameters for functions. 
  - Locals are essentially used like variables in programming languages.
  - There are GitHub issues on this, it seems to be locals and variables were one language feature many years ago and got split up and recieved additional features and the names stuck.

### Variables: inputs to a module (or just "terraform" for now)

Variables allow callers (users of Terraform) to customize behavior. We call the values they change parameters and the process itself is parameterization. 

For example, we have a deployment with an ASG. We have 2 enviroments. The dev enviorment is often broken or wastefully used by developers testing things. It does not need entrprise grade compute or memory. To optmize the speed of deployment and cost we use t3.micros for the ASG. In production we use g5.2xlarge instances. By setting a variable and replacing the aws_instance's instance type argument's **expression** from something like a literal ("t2.micro") to `var.instance_type` and adding the below code when we run `terraform apply` terraform will ask use to give it an instance type. 

```hcl
variable "instance_type" {
  description = "ASG Instance type"
  type        = string
}
```

Common arguments:
- `description`: This is of course a string to explain what this variable is for, to provide self-documentation. 
- `type`: This is the data type, more on this later. 
- `default`: This is what we want it to default to and it is optional. 
- `sensitive`: If we want to prevent this from being displayed in the terminal however it doesnt do much else, better options these days. 
- `validation`: we can control what values beyond data types are used, more on this later. 

Docs:  
https://developer.hashicorp.com/terraform/language/values/variables

---

### Locals: internal computed or repeated values

Locals are **derived values** used to avoid repetition and improve readability.

```hcl
locals {
  normalized_name = lower(var.course_name)
}
```

Use locals when:
- a value is derived from other values (an **expression** not including a literal is used essentially)
- the same expression appears multiple times
- you want clearer intent

Docs:  
https://developer.hashicorp.com/terraform/language/values/locals

---

## 3. Variable types 

Note we have 3 main groups of variable types. 
- Primitives (the simplest kind)
- Collections (groups of primitives basically)
- Complex types (some others we wont cover today plus using collections and primitives in other collections)

### string
Needs to have quotes, can have essentially any type of charachter, spaces, etc

```hcl
type = string
```

https://developer.hashicorp.com/terraform/language/expressions/types#string

---

### number
Use this for numberical values to include signed integers and floats (decimal point). For nerds: this is an arbritary precision 64 bit float but supposedly up to 512 bit. 

```hcl
type = number
```

https://developer.hashicorp.com/terraform/language/expressions/types#number

---

### bool
This is for true or false. This are **reserved** keywords meaning to terraform they have special meaning hence there is no quotes. 


```hcl
type = bool
```

https://developer.hashicorp.com/terraform/language/expressions/types#bool

---

### list(<type>)
Ordered collection of a single or any type. The elements are the members of the list. The indicies are the positions of the elements. The length is the number of elements. 

This example if of type `list(string)`. The list value is `[ "a", "b", "c" ]`. The **elements** are "a", "b", and "c". The index is the position the element is in said list. The index starts at 0. For example element "a" is at index 0 and element "c" is at index 2. The length is 3. 



https://developer.hashicorp.com/terraform/language/expressions/types#list

---

### map(<type>)
Unordered key/value pairs. All values in a map must be of the same type. 

```hcl
{
  key1 = "value1"
  key2 = "value2"
}
```

is an example of a `map(string)`. 

https://developer.hashicorp.com/terraform/language/expressions/types#map

---

### object({...})
Strongly-typed structure with named attributes. It allows mixing of data types. In some ways (such as having key/values) it is similar to a map however the use cases for both are fairly distict. 

```hcl
type = object({
  class  = string
  teacher_email = string
  class_size = number
})
```

https://developer.hashicorp.com/terraform/language/expressions/types#object

---

## 4. Defining variable values at runtime

### Using `-var`

```bash
terraform apply -var="course_name=Terraform 201"
```

https://developer.hashicorp.com/terraform/cli/commands/apply#var-option

---

### Using `.tfvars`

Create a file:

```hcl
course_name = "Terraform 201"
is_recorded = false
```

Apply it:

```bash
terraform apply -var-file="demo.tfvars"
```

Auto-loaded filenames:
- `terraform.tfvars`
- `*.auto.tfvars`

*note:* .tfvars are considered secrets and *should not* be checked into git. 

https://developer.hashicorp.com/terraform/language/values/variables#variable-definitions-tfvars-files

---

## 5. Outputs and accessing data

Outputs expose values after evaluation. They are saved into the state and have a variety of uses. 

```hcl
output "course_name" {
  value = var.course_name
}
```


https://developer.hashicorp.com/terraform/language/values/outputs

---

## 6. Conditionals (ternary operator)

Terraform uses a **ternary expression**, not `if` blocks.

Syntax:

```hcl
condition ? value_if_true : value_if_false
```

Example:

```hcl
output "recording_status" {
  value = var.is_recorded
    ? "This class is recorded."
    : "This class is not recorded."
}
```

Rules:
- `condition` must be boolean
- both result branches must be the **same type**

https://developer.hashicorp.com/terraform/language/expressions/conditionals

---

## 7. Splat operator (`[*]`)

The splat operator extracts the same attribute from **each element in a list**.

```hcl
var.students[*].name
```

Used when you have:
- list of objects
- want one attribute as a list

https://developer.hashicorp.com/terraform/language/expressions/splat

---

## 8. Arithmetic expressions

Terraform supports standard arithmetic:

- `+` addition
- `-` subtraction
- `*` multiplication
- `/` division
- `%` modulo

```hcl
output "students_missing" {
  value = var.total - var.present
}
```

https://developer.hashicorp.com/terraform/language/expressions/operators

---


## 9. Common functions
https://developer.hashicorp.com/terraform/language/functions

### Map-focused functions

- `keys(map)` → returns the keys of a map as a list
- `values(map)` → returns the values of a map as a list
- `length(map)` → returns of number of k/v pairs in a map as a number
- `merge(map1, map2, ...)` → combines two or more maps and returns a single map


---

### List & set functions

- `contains(list, value)` → checks if a value is an element of a list (or set) and returns a bool
- `length(list)` → returns the number of elements of a list as a number
- `concat(list1, list2, ...)` → combines two or more lists and returns a single list
- `sort(list)` → sorts a list alphanumerically and returns a list


---

### Boolean & numeric functions

- `anytrue(list(bool))`
- `alltrue(list(bool))`
- `abs(number)`


---

## 10. Type conversions

Terraform is strongly typed (it always knows the type). Conversions are explicit (you must do this). However providers can coerce (forefully convert values). 
Common conversions:
- `tostring`
- `tonumber`
- `tobool`
- `tolist`
- `toset`
- `tomap`

```hcl
output "port_number" {
  value = tonumber(var.port_string)
}
```



---

## 11. Variable validations

Validations fail **early** with helpful messages.

```hcl
validation {
  condition     = var.max_students >= 1
  error_message = "Must be at least 1."
}
```

Common uses:
- range checks
- allowed values
- non-empty strings

https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules

---



