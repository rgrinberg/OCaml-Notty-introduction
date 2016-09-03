# Ocaml Notty library

## Introduction

Better than curses/ncurses, here is Notty. Written in OCaml, this library
is based on the notion of composable images.

## Basics

### Image

An image is rectangle displayed in the terminal that contains styled characters.
An image can be a single character with display attributes, or some text or
a combinaison of both beside, above or over each other.

After its construction, the image can be rendered, basically it is tranformed
to a string we can display.

Print a red "Wow!" above its right-shifted copy:

```
open Notty
open Notty_unix

(* build with
 * ocamlbuild -pkg notty -pkg notty.unix basics_wow.native
 *)
let () =
	let wow = I.string A.(fg lightred) "Wow!" in
	I.(wow <-> (void 2 0 <|> wow)) |> Notty_unix.output_image_endline
```
In this program we create an Image that is based on a string "Wow!" with some
attributes `A.(fg lightred)`. Then We compose a bigger image and display twice
the `wow` image at different position. The function `Notty_unix.output_image_endline
allow us to display the generated image.

#### The Image module:

https://pqwy.github.io/notty/Notty.I.html

##### Image Creation

Basics images can be created with :

*  `I.string` : require an attribute (style) and a string
*  `I.uchars` : require an attribute and an array of unicode value
*  `I.char`   : require an attribute, a char and 2 int for the width and the height of the grid.
*  `I.uchar`  : same as `I.char` but for unicode value.

or 2 specials primitives:

*  `I.empty`  : which is a zero sized image.
*  `I.void`   : require a width and an height, it is a transparent image.

###### I.string basic example

```ocaml
open Notty
open Notty_unix
(* ocamlbuild -pkg notty -pkg notty.unix basics_I_string.native *)
let () =
 I.string A.(fg lightred) "Wow!" |> Notty_unix.output_image_endline
```

###### I.uchars basic example

```ocaml
open Notty
open Notty_unix
(* ocamlbuild -pkg notty -pkg notty.unix basics_I_uchars.native *)
let () =
  let my_unicode_chars = [|0x2500; 0x2502; 0x2022; 0x2713; 0x25cf; 0x256d;
                          0x256e; 0x256f; 0x2570; 0x253c|] in
   I.uchars A.(fg lightred) my_unicode_chars |> Notty_unix.output_image_endline
```

###### I.char basic example

```ocaml
open Notty
open Notty_unix
(* ocamlbuild -pkg notty -pkg notty.unix basics_I_char.native *)
let () =
   I.char A.(fg lightred) 'o' 4 4 |> Notty_unix.output_image_endline

```

###### I.uchar basic example

```ocaml
open Notty
open Notty_unix
(* ocamlbuild -pkg notty -pkg notty.unix basics_I_uchar.native *)
let () =
   I.uchar A.(fg lightred) 0x2022 4 4 |> Notty_unix.output_image_endline
```

#### Image composition

There are 3 composition modes which allow you to blend simple images into
complexe ones.

*  (<|>) : puts one image after another
*  (<->) : puts one image below another
*  (</>) : puts one image on another.

###### Side by side images

```ocaml
open Notty
open Notty_unix

(* ocamlbuild -pkg notty -pkg notty.unix basic_I_side_by_side *)
let () =
  let bar = I.uchar A.(fg lightred) 0x2502 3 1 in
  let img1 = I.string A.(fg lightgreen) "image1" in
  I.(img1 <|> bar) |> Notty_unix.output_image_endline
```

###### Image above another

```ocaml
open Notty
open Notty_unix

(* ocamlbuild -pkg notty -pkg notty.unix basic_I_above_another.native *)
let () =
  let bar = I.uchar A.(fg lightred) 0x2502 3 1 in
  let img1 = I.string A.(fg lightgreen) "image1" in
  I.(img1 <-> bar) |> Notty_unix.output_image_endline
```

###### Image overlay

```ocaml
open Notty
open Notty_unix

(* ocamlbuild -pkg notty -pkg notty.unix basic_I_overlay.native *)
let () =
  let bar = I.uchar A.(fg lightred) 0x2502 3 1 in
  let img1 = I.string A.(fg lightgreen) "image1" in
  I.(img1 </> bar) |> Notty_unix.output_image_endline
```

#### The Term module

*  http://pqwy.github.io/notty/Notty_unix.Term.html

It is an helper for fullscreen, interactive applications.

##### Simple interactive fullscreen

```ocaml
(* ocamlbuild -pkg notty -pkg notty.unix basics_Term_simple_terminal.native *)
open Notty
open Notty_unix
open Common

let rec main_loop t =
  let img = I.(string A.(bg lightred ++ fg black) "This is a simple example") in
    Term.image t img;
    match Term.event t with
    | `End | `Key (`Escape, []) | `Key (`Uchar 67, [`Ctrl]) -> ()
    | `Resize _ -> main_loop t
    | _ -> main_loop t

let () =
  let t = Term.create () in main_loop t
```
This little programm just draw in all the terminal, add a string and wait for
key events. Press "Esc" and the program exits.

#### Handling the terminal size and the resize events.

```ocaml
open Notty
open Notty_unix
open Common

(* ocamlbuild -pkg notty -pkg notty.unix basics_Term_simple_terminal_size.native *)

let grid xxs = xxs |> List.map I.hcat |> I.vcat

let outline attr t =
  let (w, h) = Term.size t in
  let chr x = I.uchar attr x 1 1
  and hbar  = I.uchar attr 0x2500 (w - 2) 1
  and vbar  = I.uchar attr 0x2502 1 (h - 2) in
  let (a, b, c, d) = (chr 0x256d, chr 0x256e, chr 0x256f, chr 0x2570) in
  grid [ [a; hbar; b]; [vbar; I.void (w - 2) 1; vbar]; [d; hbar; c] ]

let rec main t (x, y as pos) =
  let img = outline A.(fg lightred ) t in
  Term.image t img;
  Term.cursor t (Some pos);
  match Term.event t with
  | `End | `Key (`Escape, []) | `Key (`Uchar 67, [`Ctrl]) -> ()
  | `Resize _ -> main t pos
  | _ -> main t pos

let () =
  let t = Term.create () in
  main t (0, 1)
```

The grid function, take a list of list of images and compose a bigger image.
The outline function draw a line a the border of the screen. The attr argument
allow to configure the style of the line.