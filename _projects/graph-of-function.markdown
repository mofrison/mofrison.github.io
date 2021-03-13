---
title:  "Graph of function"
repository: graph-of-function
preview: /assets/images/projects/graph-of-function/graph-3.jpg
excerpt: The program accepts data from the user, calculates the values of the formula Y(X), outputs the values as a table and builds a graph...
date: 2017-01-26
categories: projects
tags: [c++, studies]
layout: project
lang: En
---

## 1. Purpose of the program

The program accepts data from the user, calculates the values of the formula: _**Y(X)=a1*sin(b1*X)+a2*sin(b2*X)+a3*sin(b3*X)**_, outputs the values as a table and builds a graph.

### 1.1. Input data
The user must enter the following parameters:
*	The starting coordinate **Х**(**Х0**) - the floating-point number
*	The end coordinate **Х**(**Хк**) - the floating-point number
*	Step(**∆Х**) - the floating-point number
*	Coefficients: **a1**, **a2**, **a3**, **b1**, **b2**, **b3** integers

### 1.2. Output data
At the output, the program will show the following results:
* Vectors with values X and Y in the form of a table with the possibility of editing
* Graph _**Y(X)**_ based on the results of the formula calculation 
* ANSI-encoded text file, where each line is separated by dots, and the coordinates in the line are separated by semicolons

## 2. Project structure

File name       | The contents of the file
----------------|-------------------------
GR_HEAD.H       | List of connected libraries, declarations of global variables, class prototype Coordinate function prototypes
GR_MAIN.CPP     | The list of connected files and the **main()** function, in which the main algorithm of the program is executed(рис. 1.)
GR_PROC.CPP     | Description of the Coordinate class and definitions of functions related to processing data of this class
GR_INTRF.CPP    | Definitions of auxiliary functions of the program display of the graph on the screen
GR_RW.CPP       | File write and read function definitions

## 3. Алгоритм программы

The algorithm of the main menu is shown below:

![Pic. 1. Algorithm](/assets/images/projects/{{ page.repository }}/algorithm.jpg "Pic. 1. Algorithm")

## 4. Compilation

The program code was written in C++ using a programming environment [Borland 3.1](http://ci-plus-plus-snachala.ru/?p=121)
The choice of this development environment was determined by the principle of "necessary and sufficient".
The result of the compilation will be a console application.

## 5. Working in the program

Let's look at the interaction with the program on a small example.

5.1. After the launch, the main menu of the program will be available.

![Pic. 2. Main menu](/assets/images/projects/{{ page.repository }}/main-menu.jpg "Pic. 2. Main menu")

### 5.2. Entering parameters
Press **«F2»** to open the window **«Enter parameters»** and enter the values **X_0**, **X_k**, **∆X**, **a_1**, **b_1**, **a_2**, **b_2**, **a_3**, **b_3**(pic.3.)

![Pic. 3. Entering parameters](/assets/images/projects/{{ page.repository }}/input-parameters.jpg "Pic. 3. Entering parameters")

Press **«Enter»**.

### 5.3. Table of results
After that, we will see a table of the values of the **Х** and **Y** functions: _**Y(X)=a1*sin(b1*X)+a2*sin(b2*X)+a3*sin(b3*X)**_

![Pic. 4. Table of results](/assets/images/projects/{{ page.repository }}/table.jpg "Pic. 4. Table of results")

The same table opens when you press **«F3»** in the main menu.
You can view the entire table using the keys **«↑»** and **«↓»**.

### 5.4. Geaph of the function
After pressing  **«Enter»** or **«Esc»**  the graph of the function _**Y(X)**_ appears.

![Pic. 5. Graph of the function](/assets/images/projects/{{ page.repository }}/graph.jpg "Pic. 5. Graph of the function")

(available by clicking **«F4»** in the main menu).
Pressing any key returns us to the main menu..

### 5.5. Editing a table
To edit the data in the main menu, press the key **«F5»**.

![Pic. 6. Editing a table](/assets/images/projects/{{ page.repository }}/table-editor.jpg "Pic. 6. Editing a table")

The coordinate that is active for editing is highlighted in red.

To select a different coordinate, press the keys **«↑»**, **«↓»**, **«←»**, **«→»** or **«Tab»**.

![Pic. 7. Editing a table 2](/assets/images/projects/{{ page.repository }}/table-editor-2.jpg "Pic. 7. Editing a table 2")

To start editing, press the **«Backspace»** key and enter the required value.

![Pic. 8. Editing a table 3](/assets/images/projects/{{ page.repository }}/table-editor-3.jpg "Pic. 8. Editing a table 3")

To save this value, press **«Enter»**. 
To exit the edit mode, press **«Enter»** again.

A new graph of the function will appear in front of us:

![Pic. 9. New graph of the function](/assets/images/projects/{{ page.repository }}/graph-2.jpg "Pic. 9. New graph of the function")

### 5.6. Save the results
To save the data to a file in the main menu, press **«F6»**

![Pic. 10. Save the table](/assets/images/projects/{{ page.repository }}/save.jpg "Pic. 10. Save the table")

As a result, a file with the coordinates of the specified graph will appear in the program directory

![Pic. 11. New file with the coordinates](/assets/images/projects/{{ page.repository }}/file.jpg "Pic. 11. New file with the coordinates")

You can edit the file in notepad, save it, and then read it.

### 5.7. Read the results
To read data from a file, in the main menu, press **«F7»**, enter the path to the file and press **«Enter»**.

![Pic. 12. Reading the file](/assets/images/projects/{{ page.repository }}/infile.jpg "Pic. 12. Reading the file")

A graph will appear in the window with the changes that we made in the file:

![Pic. 13. Updated graph of the function](/assets/images/projects/{{ page.repository }}/graph-3.jpg "Pic. 13. Updated graph of the function")

## 6. Exit

By pressing the **«F12»** key, the program is terminated normally.


_Thank you for your interest! :)_
