# The Quest for an editor to last the rest of my life
## - ranting about editors and endless configuration

Welcome to, what I am guessing at will be the first of many installments, of editor ranting! In this post I will talk about Emacs, Electron based editors, wonderful self-inflicted torment by configuration of editors. 

The main goal of these blogposts are to rant about Emacs as much as possible. No, really this is probably what is going to happen. Hopefully in between rants I will find what I am looking for in a editor and then be able to use one for the rest of my life (I am very optimistic about this). I am not very optimistic about this endeavour. 

First of all, we need to have some criterion that I can try to check off when trying to find an editor for me. These are based on what I actually use and feel like I am missing when I am not able to perform these actions. There are probably hundreds of actions that I do maybe once or twice week/month that I am missing and probably will not be able to perfom in any sensible way but covering exactly everything is maybe a bit too much.

Requirements (what I actually use and need)
 - Syntax highlighting
 - Autocomplete (external/internal apis)
 - Rename refactor
 - Goto definition
 - Jump to symbol
 - Find symbol
 - Open file in project
 - Stable = Not broken most of the time 
 - Maintained 
 - Been here for a while = Will stay here for a while
 
 After giving Emacs (via Spacemacs) a shot I believe to have found the answer to why Electron editors exists. I know the last sentence took you from the early days in Bell labs in the 70s to the sluggish response time of Atom in a blink of an eye but listen closely young one. The main reason I love Emacs (rather Spacemacs) is the fact that the foundation is ancient. It will probably continue living after I am dead. That is saying something. Therefore all the time spent learning it will be of use since I can almost bet my life that Emacs (in some form) will still be here after I am not. It also helps that Emacs is pretty extensisble and thus able to adapt to the current times. This is also the main attraction of Electron based editors. They are extensible by nature given their web ancestory. Extensions, or plugins, are just some javascript (which everyone knows) just like everything in Emacs is just elisp (that no one truly knows). Electron editors are just the Emacs of our time and given the fact that Emacs continues to thrive to this day I believe electron editors such as Atom and Visual Studio Code will as well. 

 # Reflections and rants about editors 
Vanilla Emacs was too much work starting from scratch. The conclusion is to just use one of the community bundles of Emacs that are popular. The most popular that I have found is Spacemacs and Doom.

 ## Spacemacs
 Spacemancs [mnemonic](https://en.wikipedia.org/wiki/Mnemonic) (yes..) keyboard based shortcuts are amazing. They are quite possibly the thing I have been missing my whole life while using computers. Coupled with Vim bindings via Evil mode their power is truly awesome. I feel like a God moving around between buffers and paragraphs all the time while my mouse sits there of to the side looking sad and most importantly unused. The main drawback that I have found, and it is a big one, is the fact that when I am navigating my brains stops thinking about the code I was working on and more on how to navigate the editor. I get the feeling this will reduce over time and muscle memory will take over a large part of this thinking and thus make my feeling like God whose brain is currently active and not trying to coordinate ten fingers moving around pressing keys like a madman. 

## Doom Emacs


### Operating system independence
Nope. Almost. Sometimes packages simply does not work on one operating system (I am looking at you Windows..). This is a major pain since one of the criterions for me is its independence. Having the same setup working exactly the same way across macOS, Linux and Windows is the wettest dream of them all. That and actually conjuring up the courage to learn Vulkan. 

 ### Autocomplete
 God lord save me from the endless list of plugins and config files.. Oh, look [LSP](https://github.com/Microsoft/language-server-protocol). Works pretty well with both C/C++ and Rust which is nice. Supports the basic and most frequently used actions I need such as 'Find all references', 'Find definition', etc. This is the best bet to just save us all from the doom that is autocomplete. There is LSP-like offshoot called debug adapter protocol [DAP](https://code.visualstudio.com/blogs/2018/08/07/debug-adapter-protocol-website/) that might solve some debugger issues as well! Hurray!

 ### Syntax highlighting
 It works! Its weird and not perfect! Hey! Just like Emacs! Fight me.

 ### Org mode
 Org mode seems so cool that I want to write my master's thesis in it but the integration with LaTeX was truly a project in of itself that I abandonded it. 

 ### Magit
 This is a weird one. Git is scary. Emacs is scary. But, together they make something that feels smooth. Not silky smooth but more like whiskey smooth. It is clearer and easier to use Git with Magit than on the commandline. I believe that the problem is that Git concepts must be adopted from usage somewhere else before one tries Magit. I like it so far atleast.

### Typing latency
Spacemacs is noticeably slower than Doom. Using black magic and shamistic rituals VS Code is actually able to have a descent typing latency (atleast in my experience) which is amazing.

## Visual Studio Code
Better than Atom. Actually works. Huge community. 

## Sublime Text 3
Fast, shitty and okay.