---
title: "Building Minesweeper with Go"
date: 2019-02-28T16:35:04-08:00
draft: false
---

After going through the [Tour of Go](https://tour.golang.org/list) I started my first toy program -- the game of Minesweeper ([on github](https://github.com/mikihau/minesweeper)). To exercise what I have gotten down with Go I had a couple of things in mind about this program before I started:

- It should be feature complete -- just all things you can do with the game.
- It should only depend on the standard library, so that I can focus on the basics.

When I finished it turned out like this:
![Demo](/images/minesweeper-demo.gif)

Pretty sweet!

As a newbie Gopher I think it'll be helpful to write down things I learned, and ideas for improvement while working on this initial version. Here they are:

1. **Every assignment creates a (shallow) copy.** I was caught by surprise while working on the recursive `explore()` function, where I want to get hold of a cell by its coordinates within the board, then modify some properties of the cell.

    This works:
    ```go
    cell := &b.grid[x][y]
    cell.revealed = true // was previously false
    ```
    This also works:
    ```go
    b.grid[x][y].revealed = true
    ```
    This doesn't work:
    ```go
    cell := b.grid[x][y]
    cell.revealed = true // was previously false
    ```
    Coming from the Python world I had assumed the third variation would have worked, because well, the the grid hold the cell object, so it should be equivalent to saying `b.grid[x][y].revealed = true`. Turned out it's not the case. Under the hood, the assignment copies the content of the cell, and gives me that copy. So whatever I do with `cell` has nothing to do with `b.grid[x][y]` because they're two seperate objects. The caveat here is that the copy made by assignment is shallow. If a cell contains references, e.g. arrays/slices/pointers/maps/functions/channels (what about structs?), then the copied struct still holds on to those same objects in those fields, because they have the references copied.

2. **Duck typing is out (or at least out of easy reach).** This issue came up when I was trying to validate user inputs from the command line. In my program, some command handlers expect string arguments, and some expect ints. If I were to do this in Python, I'd be happy to write a function that returns a list of arguments (of the type that the handler expect) after sanitizing the inputs. But in Go _you can't return a slice of an undetermined type_, regardless of type homogeneity within that slice. This really calls out for generics, which Go intentionally doesn't offer (for good reasons!). Without generics I suspect it can probably be done with some sort of `interface{}` and/or reflection, but fighting with the type system usually means there are better ways to do it -- I'll be on the look out.

3. **Abstractions with structs + interfaces vs classes.** From what I get from the Tour of Go, a struct is a collection of fields, and an interface is a collection of method signatures. It came as an afterthought to me that if I combine these two features together, I get what I would normally expect for a class as an OOP language. As an example, in the code I have this lengthy switch statement taking care of all argument validation, and executing commands. What I could have done is to abstract the command as an interface, and each command (`new`, `reveal`, `flag`, `exit`, and `help`) gets to keep its own set of fields, and have its own behavior for validation, execution and exit.
    ```go
    type command interface {
        validate func(string) ([]string, error)
        execute  func(*board, []int)
        exit     func(board)
    }
    ```
    For example, the `reveal` command can be defined as the following:
    ```go
    type revealCommand struct {
        numArgs int
        args    []int
    }

    func (r revealCommand) validate(input string) error {
        parsedArgs := strings.Fields(input)
        if len(parsedArgs) != r.numArgs {
            return fmt.Errorf("Wrong number of arguments: expecting %v, got %v -- type 'h' for help", r.numArgs, len(parsedArgs))
        }

        for i, arg := range parsedArgs {
            if value, err := strconv.Atoi(arg); err == nil {
                r.args[i] = value
            } else {
                return fmt.Errorf("Expecting integer arguments -- type 'h' for help")
            }
        }
        return nil
    }

    func (r revealCommand) execute(b *board) {
        b.updateOnReveal(r.args)
    }

    func (r revealCommand) exit(b board) {
        fmt.Println(b)
    }
    ```
    This set up simplifies the main eval loop. After instantiating a command, all we have to do  is to call `validate()`, `execute()` and `exit()` on it.

4. Spice it up with goroutines? No thanks, not for minesweeper. In this game, all board updates are just changing some bits in memory, so it can happen synchronously. Plus we're constantly waiting on user inputs, one move at a time -- there can be better situations where concurrency is actually useful.