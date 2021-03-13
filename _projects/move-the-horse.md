---
title: Move the Horse
repository: move-the-horse
preview: ./assets/images/projects/move-the-horse/Greeting.jpg
excerpt: This training program allows you to simulate the movement of a chess piece «Horse» on a field of size 8x8. An additional condition is that the «Horse» must visit each cell only once...
date: 14-06-2016
categories: projects
tags: [c++, studies]
layout: project
lang: En
---

## 1. Purpose of the program

This training program allows you to simulate the movement of a chess piece **«Horse»** on a field of size 8x8.
An additional condition is that the **«Horse»** must visit each cell only once.
_**Attention! The program was tested only in the environment [Borland 3.1](http://ci-plus-plus-snachala.ru/?p=121)!**_

## 2. Project structure

File name       | The contents of the file
----------------|-------------------------
HORSEPRT.H      | Connecting standard libraries, declaring variables, classes, methods, and functions
HORSE.CPP       | The main project file, contains the function **main()**
HORSECTR.CPP    | Description of the program management functions
HORSERND.CPP    | Description of methods and functions for finding the best crawl path
HORSEOUT.CPP    | Dialog functions, user interface
HORSEVIZ.CPP    | Visualization of the chess field
HORSEGPR.CPP    | Description of methods and functions for finding the best traversal path with visualization of the chessboard
Help.txt        | File with hints

## 3. Compilation

To compile and run, it is best to use a simple one, like an Ilyich Light Bulb, and reliable as a Kalashnikov Assault Rifle, [Borland 3.1](http://ci-plus-plus-snachala.ru/?p=121)
The choice of this development environment was determined by the principle of «necessary and sufficient».
The result of the compilation will be a console application.


## 4. Starting the program

After running the compiled exe file, a splash screen will appear in a new window, as shown in the figure below(Pic. 1.):

![Pic. 1. Screen saver](/assets/images/projects/move-the-horse/Greeting.jpg?raw=true "Pic. 1. Screen saver")

## 5. Main menu

After the launch, the main menu of the program will be available.

![Pic. 2. Main menu](/assets/images/projects/move-the-horse/Main-menu.jpg?raw=true "Pic. 2. Main menu")

As you can see in the figure above (Pic. 2.), the main menu contains 7 items,which are selected using the function keys(**F1**-**F6**, **F10**).
Let's look at these points in more detail:

## 5.1. Help

When you press the F1 key in the main menu, the program displays the contents of the file: _**Help.txt**_

![Pic. 3. Help](/assets/images/projects/move-the-horse/Help.jpg?raw=true "Pic. 3. Help")

## 5.2 Я - и есть конь(I am the horse)

When you press the F2 key, a chessboard with the size of 8x8 cells will appear in the program window(pic. 4.)

![Pic. 4. Chessboard](/assets/images/projects/move-the-horse/Manual-mode-bypass.jpg?raw=true "Pic. 4. Chessboard")

The user can independently, by clicking the mouse, select the cells on which the figure "horse"should move.
It can also switch to automatic mode by clicking on the field labeled **«Руч»** in the upper-right corner.
After that, the label in this field will change to **«АВТ»** and after selecting the next cell, the program will start moving around the field on its own(pic.5.).

![Pic. 5. From manual to automatic mode](/assets/images/projects/move-the-horse/The-transition-from-manual-to-automatic-mode.jpg?raw=true "Pic. 5. From manual to automatic mode")

## 5.3. Хромая кобыла(The lame mare)

When press **«F3»**, the user will be prompted to enter the coordinates of the initial position of the chess piece "knight"
After that, an automatic crawl of the entire field will start from the specified position.
At the same time, the user will be able to track not only the current position, possible paths of movement or the completed trajectory,
but also the methods and functions called in the program(pic. 6.).

![Pic. 6. Automatic mode on the chessboard](/assets/images/projects/move-the-horse/Semi-automatic-mode.jpg?raw=true "Pic. 5. Automatic mode on the chessboard")

In the event of deadlocks (when not all the cells have been visited yet, but there is nowhere to step from the current position), the program returns to the previous fork, where the program changes the path of the bypass (pic. 7.).

![Pic. 7. Return to the fork](/assets/images/projects/move-the-horse/Rollback.jpg?raw=true "Pic. 5. Return to the fork")

## 5.4. Реактивный конь(Jet Horse)

When you press the **«F4»** key, the program independently sets the initial position of the "Horse" figure, bypasses the entire chessboard, moves one square and starts the crawl again. And so on until it reaches the last cell.

## 5.5. Посмотреть пройденный путь(View the completed path)

By pressing the **«F5»** key, the program will display the trajectory of the last round of the chess field in the form of a sequence of coordinates(pic. 8.):

![Pic. 8. The trajectory of the completed path](/assets/images/projects/move-the-horse/The-path.jpg?raw=true "Pic. 8. The trajectory of the completed path")

## 5.6. Записать траекторию в файл(Write the trajectory to a file)

By pressing the **«F6»** key, the program will create an array and initialize it with the coordinates of the last crawl path(pic. 9.):

![Pic. 9. Initializing an array](/assets/images/projects/move-the-horse/Initialize-array-for-output-trajectory-file.jpg?raw=true "Pic. 9. Initializing an array")

After that, it will ask the user to enter a file name to write the trajectory to the file(pic. 10.):

![Pic. 10. Write the trajectory to a  file](/assets/images/projects/move-the-horse/Save-trajectory-to-file.jpg?raw=true "Pic. 10. Write the trajectory to a  file")

The result of writing the trajectory to a file(pic. 11):

![Pic. 11. Trajectory](/assets/images/projects/move-the-horse/track.jpg?raw=true "Pic. 11. Trajectory")

##5.7. Shutting down the program

By pressing the **«F10»** key, the program is normally terminated.

_Thank you for being able to read this to the end! :)_
