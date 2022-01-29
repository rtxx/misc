# Getting your i3 with ‘togglable’ light and dark themes 2 : The Revengeance

Welcome, one and all! Sit in your cozy place and bring your snacks, it's learning time!

On our latest installment, we started working on our script, to automagicaly change our themes. Now we are going to improving it, by adding a bunch of new programs, more functionality and error proofing but try to stay simple along the way. The goal is to have a simple script that we can change easly and adapt it to our liking.

The script working

Let's get ready to ruuuumble...

# First things first
This time around, not only we are change GTK and Qt theme but also:

- kitty
- Xresources
- dunst
- wallpaper
- i3wm
- i3status

We are also adding functionality:

- Get a list of the themes avaiable and **valid**
- Check if our JSON values are valid
- Check if our theme files are present
- Check if our theme file structure is valid (more on this later)
- Only updating when all the above are **true**
- And finally, be more flexible:
	+ Want to change only GTK and Qt? Done.
	+ Want to change GTK from one theme and i3wm, i3status and dunst from another? Done and done.
	+ Meaning: make our script more *modular*
	
So without further ado, let's begin!

# Beefing up our script

Note: I will only put here some snippets of the code, so its more clear and we dont get a wall of code. All the code from the script is commented. I will only explain what I think is not super straight foward.

## Add flags
The 1st thing that I like to do in my scripts is allowing for flags. It adds more flexabillity to the user and we can, for example, add a help text which is *very* useful.
```
# check if dependencies are met
dependencies=("xsettingsd" "jq" "feh")
for pkg in ${dependencies[@]}; do
...
done

function printHelp {
cat << EOF
...
EOF
}

# prints help if no argument is present
if [[ $# -eq 0 ]]; then
...
fi

#loops the argument string until is done
while [[ $# -gt 0 ]]; do
  key="$1"
  case "$key" in
    -h|--help) printHelp=1 ;;
    -l|--list) listThemesF=1 ;;
    -a|--apply) shift; applyTheme="$1" ; applyThemeF=1 ;;
    *) echo "Unknown option: '$key' (use -h for help)" ; exit 2 ;;
  esac
  shift
done

# if help flag is true
if [ "$printHelp" ]; then
...
fi
```
Letś break it down.

- ```for pkg in ${dependencies[@]}; do``` This check for the dependencies that our script needs. Note: This only works on Arch by the way. If using other distro, remove it or change it to something that check for packages installed by your package manager. If you remove it, please make sure you have all the packages installed before running!

- ```function printHelp``` its a simple [HERE](https://en.wikipedia.org/wiki/Here_document) funcion from ```bash```.

- ```if [[ $# -eq 0 ]]; then``` is for when the script is executed without any flags. Outputs ```function printHelp```.

- ```while [[ $# -gt 0 ]]; do``` loops through the argument string.
	+ ```-a|--apply) shift; applyTheme="$1" ; applyThemeF=1 ;;```
	+  ```applyTheme``` gets the theme name ie *Breeze* and ```applyThemeF``` is just a flag, so we now that the flag ```-a, --apply``` is active.
- ```if [ "$printHelp" ]; then``` outputs ```function printHelp```

## JSON checking
Next up is some love for our JSON file
```
# checks if JSON files is valid
# empty files will NOT error out
function verifyJSON() {
...
}

# list all the themes avaiable
if [ "$listThemesF" ]; then
  echo "Themes available:"
  #checks for theme.config within themes folder
  for currentThemeDir in $themesDir/*
  do
    # checks if theme.config file exist and verifies if its a valid JSON file
    if [[ -e $currentThemeDir/theme.config ]]; then
      if verifyJSON "$currentThemeDir/theme.config"; then
        echo "--> $(basename $currentThemeDir)"
      else
        echo "--> Invalid JSON: $(basename $currentThemeDir)"
      fi
    fi
  done
  exit 0
fi
```
```function verifyJSON()``` checks if our JSON is valid but **not empty**. This is important, because if we create a ```theme.config``` and its empty, ```jq``` will **not** give us an error. To evade problems, when we get a ```jq``` value from our config file, we **always** check it if it's **not null** (more on this later).

```if [ "$listThemesF" ]; then``` lists all the themes avaiable. Let's break it down:
- Scans the themes dir, and looks for ```theme.config```
- If it exists, then verifies if it is a valid JSON file
	+ if true, then ouptuts the name of the folder with ```basename```
	+ if false, then outputs 'Invalid JSON'

And like I said before, if the ```theme.conf``` is **empty** it will say as if it is a **valid** theme.

## GTK and Qt (again)
