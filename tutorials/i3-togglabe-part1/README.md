# Getting your i3 with ‘togglable’ light and dark themes
So you have a lean, mean machine with i3 installed and, like me, you like to use a light and dark theme and noticed that when using, for example, Firefox, some websites have a dark mode but won’t change it automatically. If this bothers you then come along. Onwards!

For this tutorial, I’m using Arch Linux BTW™ with i3 installed with no DM, but I’m sure you can apply it to other WM’s.

We are going to need a few packages:

* **xsettingsd** – Lightweight xsettings daemon
* **lxappearance-gtk3** – GTK+ theme switcher
* **qt5ct-kde** – Qt5 configuration utility
* **breeze-gtk** – GTK theme with light and dark colors, you can use others to your liking.
* **breeze** – Qt theme with light and dark colors, but it’s a bit ‘heavy’, use another if you like. breeze has the benefit of having a GTK and Qt themes, hurray for consistency!
* **papirus-icon-theme** – Pretty on the eyes icon theme
* **capitaine-cursors** – Stylish cursor theme
* **noto-fonts** – Good looking fonts from Google
* **jq** – JSON parser

[IMAGE]
The finished script
# Getting GTK to work

Let’s start with GTK applications. ```xsettingsd``` will help us do this. For more info on what it does check this, this, and this. Let’s config it. [needs url's]

Start ```xsettingsd```. Create a new file on your home folder and name it **.xsettingsd**. Open it and write:
```
Net/ThemeName "Breeze"
```
Save it. To reload it use ```killall -HUP xsettingsd```. This tells ```xsettingsd``` to read and reload according to the file we just created. So to change to another theme, open it and change **Net/ThemeName** to the theme you want, in this case, **Breeze-Dark**
```
Net/ThemeName "Breeze-Dark"
```
Save it and run again ```killall -HUP xsettingsd```. The GTK theme will change, and all the open GTK applications will follow. Nice! But this is very annoying if you want to change it so often. To ease our life, we can change it with sed:
```
sed -i '/Net\/ThemeName/cNet\/ThemeName 'Breeze'' ~/.xsettingsd
```
So we can make something like
```
#!/bin/bash
sed -i '/Net\/ThemeName/cNet\/ThemeName '$1'' ~/.xsettingsd
killall -HUP xsettingsd
```
Save it as i3theme on ~ (for example) and run it:

For light theme:
```
bash ~/i3theme \"Breeze\"
```
For dark theme:
```
bash ~/i3theme \"Breeze-Dark\"
```

or create an i3 hotkey like this:
```
bindsym $mod+F1 exec bash ~/i3theme '"Breeze"'
bindsym $mod+Shift+F1 exec bash ~/i3theme '"Breeze-Dark"'
```
Tada! Now we have a semi-automatic, semi ‘togglable’ way of changing our GTK theme.

How about Qt apps? And apps like ```xterm``` that use Xresources? Or ```dunst``` (notification manager)? Or ```i3``` / ```i3status``` colors? What about other stuff like icons and cursors? Yes, that’s quite a lot of stuff that needs changing. But the good news is where there’s a will there’s a way.

# Getting Qt to work

To change Qt theme, we can use ```qt5ct```. But the problem is that the version on Arch official repositories can’t change the color scheme. AUR to the rescue then! There is a package on AUR called ```qt5ct-kde``` that has the functionality that we need. Sweet! Now, all we have to do is config it to our liking.

Let’s begin by enabling ```qt5ct```. Open ```/etc/environment``` in your favorite text editor (don’t forget you need sudo / doas) and type:
```
QT_QPA_PLATFORMTHEME=qt5ct
```
Save, relog, and ```qt5ct``` in now active.

Cool, now what? Now we do the same process as above for the GTK themes. There’s a file located on **~/.config/qt5ct/qt5ct.conf** that holds the info we need (if not open ```qt5ct``` and hit apply). Because *nix loves text, we can change it with sed again, neat!
```
#!/bin/bash
# GTK related
sed -i '/Net\/ThemeName/cNet\/ThemeName '$1'' ~/.xsettingsd
killall -HUP xsettingsd

# Qt related
sed -i '/style/cstyle='"$1"'' ~/.config/qt5ct/qt5ct.conf
sed -i '/color_scheme_path/ccolor_scheme_path='"/usr/share/color-schemes/$1.colors"'' ~/.config/qt5ct/qt5ct.conf
```
Save it and try it. Now we can change GTK and Qt in one fell swoop! Awesome!

Well not really. If you run the above script it will not work properly *sad face*. Because we are passing ```\"Breeze\"``` with quotations in which the ```qt5ct``` config file does not need, but the ```xsettingsd``` **needs**.

We could add a 2nd argument to the script, but it seems a bit tiresome. We could also simply not change the style and use only breeze and change only the color scheme. But I’m not too fond of the quotations in the middle of the color scheme path… And this problem is only getting worse as we add other programs into the mix. So let’s try to tackle it now.

We can have several different scripts and hard code them. But I don’t like that, it seems so *not* elegant. Instead, I think a nice way of doing this is to have a file that holds all the info we need per theme. So we can have a directory tree like this:

    i3theme (our script)
    themes
        breeze
            theme.config
            wallpaper.jpg
            etc…
        breeze-dark
            theme.config
            wallpaper.jpg
            etc…

That **theme.config** is the file that has the info we need, including:

* GTK theme
* Qt theme
* Wallpaper
* ```dunst``` theme
* ```i3``` config
* and other stuff you like to change

**Let’s get to work.**
```
#!/bin/bash
# get the name of the script from its file name
scriptName=$(basename $0)
#https://stackoverflow.com/a/246128
scriptDir="$( cd -- "$( dirname -- "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )"
themesDir=$scriptDir/themes

# sets the xsettingsd and qt5ct file paths
xsettingsdConfigFile=~/.xsettingsd
qtConfigFile=~/.config/qt5ct/qt5ct.conf

# using jq to get the info we need
# getting JSON file contents to var
themeJSON=$(cat $themesDir/$1/theme.config)

# GTK values from JSON
gtkThemeName=$( echo $themeJSON | jq -r '.gtk["gtk-theme-name"]' )
gtkThemeNameQ='"'$gtkThemeName'"' # added quotation marks

# Qt values from JSON
qtThemeName=$( echo $themeJSON | jq -r '.qt["style"]' )
qtColorSchemeName=$( echo $themeJSON | jq -r '.qt["color_scheme_path"]' )

# GTK theme updating section
sed -i '/Net\/ThemeName/cNet\/ThemeName '$gtkThemeNameQ'' $xsettingsdConfigFile

# on the fly theme change for GTK apps, reloading xsettingsd
killall -HUP xsettingsd

 # Qt theme updating section
sed -i '/color_scheme_path/ccolor_scheme_path='"$qtColorSchemeName"'' $qtConfigFile
sed -i '/style/cstyle='"$qtThemeName"'' $qtConfigFile

exit
```
breeze – theme.config
```
{
  "gtk":
    {
      "gtk-theme-name": "Breeze"
    },
  "qt":
    {
      "color_scheme_path": "/usr/share/color-schemes/Breeze.colors",
      "style": "Breeze"
    }
}
```
breeze-dark – theme.config
```
{
  "gtk":
    {
      "gtk-theme-name": "Breeze-Dark"
    },
  "qt":
    {
      "color_scheme_path": "/usr/share/color-schemes/BreezeDark.colors",
      "style": "Breeze"
    }
}
```
Damn, that’s a lot of code for such a simple task! But it’s what we said we were going to do. It’s basically the same as before, but now we get all the info we need from ```theme.config``` which is a **JSON** file. That’s where ```jq``` comes to play. Please review the code and the comments, it’s pretty self-explanatory.

To use the script do the following,

* Create a folder named themes and inside create 2 new folders:
	* breeze and breeze-dark.
* Inside each one, put their corresponding **theme.config**
* Our directory tree should look something like this:
```
i3theme (our script)
 themes
     breeze
         theme.config
     breeze-dark
         theme.config
```
Now we only need to do ```bash i3theme [name of the theme which is the name of the directory]``` i.e:
```
bash i3theme breeze
bash i3theme breeze-dark
```
Hot damn, we did it! Now we have a base that we can build upon with ease.

# Turbocharging our script

Now that we have the basis of our script done, we can spice it up. If you read about ```xsettingsd```, you noticed that we can change more than the GTK theme. We can change the theme, the icons, the mouse theme, and even some font settings. For our purpose, we need two more settings, **Net/IconThemeName** and **Gtk/CursorThemeName**. We also need to change two more files: **~/.config/gtk-3.0/settings.ini** and **~/.gtkrc-2.0**. Why? Because that’s the files that ```lxappearance``` use for setting GTK 2/3 theme. From this, we also need

 * Icon name
 * Font name
 * Cursor theme name and size

We also added ```icon_theme``` to Qt theme.

> Side note: I know that lxappearance complains about to not edit directly ```gtkrc-2.0```, but I never had any problems, just remember that if you use ```lxappearance```, all the configs made by our script will be gone!

Let’s add the changes to our script:
```
#!/bin/bash
# get the name of the script from its file name
scriptName=$(basename $0)
#https://stackoverflow.com/a/246128
scriptDir="$( cd -- "$( dirname -- "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )"
themesDir=$scriptDir/themes

# sets the xsettingsd and qt5ct file paths
xsettingsdConfigFile=~/.xsettingsd
qtConfigFile=~/.config/qt5ct/qt5ct.conf

# sets GTK config files that lxappearance uses
gtk3ConfigFile=~/.config/gtk-3.0/settings.ini
gtk2ConfigFile=~/.gtkrc-2.0

# using jq to get the info we need
# getting JSON file contents to var
themeJSON=$(cat $themesDir/$1/theme.config)

# GTK values from JSON
gtkThemeName=$( echo $themeJSON | jq -r '.gtk["gtk-theme-name"]' )
gtkThemeNameQ='"'$gtkThemeName'"' # added quotation marks
gtkIconName=$( echo $themeJSON | jq -r '.gtk["gtk-icon-theme-name"]' )
gtkIconNameQ='"'$gtkIconName'"' # added quotation marks
gtkCursorName=$( echo $themeJSON | jq -r '.gtk["gtk-cursor-theme-name"]' )
gtkCursorNameQ='"'$gtkCursorName'"' # added quotation marks
gtkCursorSize=$( echo $themeJSON | jq -r '.gtk["gtk-cursor-theme-size"]' )
gtkFontName=$( echo $themeJSON | jq -r '.gtk["gtk-font-name"]' )

# Qt values from JSON
qtThemeName=$( echo $themeJSON | jq -r '.qt["style"]' )
qtColorSchemeName=$( echo $themeJSON | jq -r '.qt["color_scheme_path"]' )
qtIconThemeName=$( echo $themeJSON | jq -r '.qt["icon_theme"]' )

# GTK theme updating section
# xsettingsd section
sed -i '/Net\/ThemeName/cNet\/ThemeName '$gtkThemeNameQ'' $xsettingsdConfigFile
sed -i '/Net\/IconThemeName/cNet\/IconThemeName '$gtkIconNameQ'' $xsettingsdConfigFile
sed -i '/Gtk\/CursorThemeName/cGtk\/CursorThemeName '$gtkCursorNameQ'' $xsettingsdConfigFile

# gtk 2.0 section
sed -i '/gtk-theme-name/cgtk-theme-name='"$gtkThemeName"'' $gtk2ConfigFile
sed -i '/gtk-icon-theme-name/cgtk-icon-theme-name='"$gtkIconName"'' $gtk2ConfigFile
sed -i '/gtk-font-name/cgtk-font-name='"$gtkFontName"'' $gtk2ConfigFile
sed -i '/gtk-cursor-theme-name/cgtk-cursor-theme-name='"$gtkCursorName"'' $gtk2ConfigFile
sed -i '/gtk-cursor-theme-size/cgtk-cursor-theme-size='"$gtkCursorSize"'' $gtk2ConfigFile

# gtk 3.0 section
sed -i '/gtk-theme-name/cgtk-theme-name='"$gtkThemeName"'' $gtk3ConfigFile
sed -i '/gtk-icon-theme-name/cgtk-icon-theme-name='"$gtkIconName"'' $gtk3ConfigFile
sed -i '/gtk-font-name/cgtk-font-name='"$gtkFontName"'' $gtk3ConfigFile
sed -i '/gtk-cursor-theme-name/cgtk-cursor-theme-name='"$gtkCursorName"'' $gtk3ConfigFile
sed -i '/gtk-cursor-theme-size/cgtk-cursor-theme-size='"$gtkCursorSize"'' $gtk3ConfigFile

# on the fly theme change for GTK apps, reloading xsettingsd
killall -HUP xsettingsd

 # Qt theme updating section
sed -i '/color_scheme_path/ccolor_scheme_path='"$qtColorSchemeName"'' $qtConfigFile
sed -i '/style/cstyle='"$qtThemeName"'' $qtConfigFile
sed -i '/icon_theme/cicon_theme='"$qtIconThemeName"'' $qtConfigFile

exit
```
breeze – theme.config
```
{
  "gtk":
    {
      "gtk-theme-name": "Breeze",
      "gtk-icon-theme-name": "Papirus",
      "gtk-font-name": "Noto Sans 11",
      "gtk-cursor-theme-name": "capitaine-cursors",
      "gtk-cursor-theme-size": "0"
    },
  "qt":
    {
      "color_scheme_path": "/usr/share/color-schemes/Breeze.colors",
      "style": "Breeze",
      "icon_theme": "Papirus"
    }
}
```
breeze-dark – theme.config
```
{
  "gtk":
    {
      "gtk-theme-name": "Breeze-Dark",
      "gtk-icon-theme-name": "Papirus-Dark",
      "gtk-font-name": "Noto Sans 11",
      "gtk-cursor-theme-name": "capitaine-cursors",
      "gtk-cursor-theme-size": "0"
    },
  "qt":
    {
      "color_scheme_path": "/usr/share/color-schemes/BreezeDark.colors",
      "style": "Breeze",
      "icon_theme": "Papirus-Dark"
    }
}
```
And that’s it, folks! We have a functional script that helps us change our theme.

For extra darkmodness install this extension: [Dark Reader](https://darkreader.org/).  After, click on the 3 little dots and click on **“Use system color scheme”**. Now even if a website doesn’t have a dark mode, we can use the Dark Reader version. Neot-o!

But it’s not the end. What about if we change to a theme that’s not installed? What about if our JSON file is not properly formatted or is empty? What about ```i3``` or ```Xresources``` colors? What about making a simple tray icon that we can easily switch from light and dark themes? So much to do… Awesome! It’s always good to have something to look forward to or improve. But it’s a story for another time. Stay tuned for a part 2 maybe even a part 3.

So long and thanks for all the fish!
