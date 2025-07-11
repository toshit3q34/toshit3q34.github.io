---
title: "Chained Unary Minus Resolution"
date: 2025-01-30
author: "Toshit Jain"
words_per_minute : 100
read_time: true
tags :
    - Open Source
    - C++
---

## Introduction and Motivation

I came to know about Open Source in July-October of 2024 and was really excited to have a contribution for myself as well. I was interested in C++ and thus landed on Clang Based Automatic Differentiation (CLAD) OS repository. I did a contribution in December 2024 to January 2025 which I will be explaining in this blog.
You can view the PR here : [PR Link](https://github.com/vgvassilev/clad/pull/1180)
Clad Repository : [Link](https://github.com/vgvassilev/clad)

---

## Clang Based Automatic Differentiation

It is a LLVM compiler infrastructure and a plugin for Clang. It enables automatic differentiation for functions in C++. Information about Automatic Differentiation can be found [here](https://en.wikipedia.org/wiki/Automatic_differentiation).

---

## Issue and naive solution

CLAD parses the code presented using Clang's Abstract Syntax Tree (AST) using LLVM libraries. The given function is parsed at compile time and a derivative is added in the object file. The issue I worked on was resolving repeated use of Unary Minuses. So, the issue was when we used chained minuses, like :

```cpp
Expression : -(-(-x))
The differentiation should be : -1
However, due to mishandling it displayed - - -1
```

Naive solution which I presented at first was to resolve it at run time (which was obviously pointed out and was wrong !!). I came to know about Clang and LLVM then and started learning the required stuff as recommended by the maintainer of the repo.

---

## Resolution Function

The following function was added.

```cpp
 Expr* VisitorBase::ResolveUnaryMinus(Expr* E, SourceLocation OpLoc) {
    if (auto* UO = llvm::dyn_cast<clang::UnaryOperator>(E)) {
      if (UO->getOpcode() == clang::UO_Minus)
        return (UO->getSubExpr())->IgnoreParens();
    }
    Expr* E_LHS = E;
    while (auto* BO = llvm::dyn_cast<BinaryOperator>(E_LHS))
      E_LHS = BO->getLHS();
    if (auto* UO = llvm::dyn_cast<clang::UnaryOperator>(E_LHS->IgnoreCasts())) {
      if (UO->getOpcode() == clang::UO_Minus)
        E = m_Sema.ActOnParenExpr(E->getBeginLoc(), E->getEndLoc(), E).get();
    }
    return m_Sema.BuildUnaryOp(nullptr, OpLoc, clang::UO_Minus, E).get();
  }
```

The function does the following things :
- Recursively traverses down the expression
- The first section adds unary minus and removes unary minus if already present, ensuring proper meaning. Example : -(-(-x)) starts as x -> -x -> x -> -x
- The second part iteratively travels down the leftmost expression if binary expression and adds Unary Minus after adding parens to the expression. Example : Unary minus on -a+b is -(-a+b) and not - -a+b = a+b (Wrong!)

This function, though seems small, deals with all the edge cases regarding Unary Minus.

---

## Unit testing

The following unit tests were added. The first one deals with the incorrect parsing in independent expressions while the second deals with incorrect parsing for Unary Minus over expressions with Binary Operators.

```cpp
// RUN: %cladclang %s -I%S/../../include -oUnaryMinus.out 2>&1 | %filecheck %s
// RUN: ./UnaryMinus.out | %filecheck_exec %s
// RUN: %cladclang -Xclang -plugin-arg-clad -Xclang -enable-tbr %s -I%S/../../include -oUnaryMinus.out
// RUN: ./UnaryMinus.out | %filecheck_exec %s

#include "clad/Differentiator/Differentiator.h"

#include "../TestUtils.h"

double f1(double x)
{
    return -(-(-1))*-(-(-x));
}

//CHECK: void f1_grad(double x, double *_d_x) {
//CHECK-NEXT:    *_d_x += -(-1 * 1);
//CHECK-NEXT: }

double f2(double x, double y)
{
    return -2*-(-(-x))*-y - 1*(-y)*(-(-x));
}

//CHECK: void f2_grad(double x, double y, double *_d_x, double *_d_y) {
//CHECK-NEXT:    {
//CHECK-NEXT:        *_d_x += -(-2 * 1 * -y);
//CHECK-NEXT:        *_d_y += -(-2 * -x * 1);
//CHECK-NEXT:        *_d_y += -1 * -1 * x;
//CHECK-NEXT:        *_d_x += 1 * -y * -1;
//CHECK-NEXT:    }
//CHECK-NEXT: }

double dx;
double arr[2] = {};
int main(){

    INIT_GRADIENT(f1);
    INIT_GRADIENT(f2);

    TEST_GRADIENT(f1, 1, 5, &dx); // CHECK-EXEC: 1.00
    TEST_GRADIENT(f2, 2, 3, 4, &arr[0], &arr[1]) // CHECK-EXEC: {-4.00, -3.00}
}
```

## Learnings

This contribution was my first ever to C++ as well as Clad. I learned a lot about Clang and LLVM. Looking forward to more contributions to Clad in the future !!