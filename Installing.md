Mira requires java 1.5 and the [trove](http://trove4j.sourceforge.net/) library (included).

  1. Download the source code using [subversion](http://subversion.tigris.org/):
```
svn checkout http://miralium.googlecode.com/svn/trunk/ miralium
```
  1. One step compilation with ant (jump to 5):
```
ant
```
  1. If you don't have ant, compile manually (in the miralium directory):
```
javac -cp trove-2.1.0.jar edu/lium/mira/*.java
```
  1. Create jar file:
```
jar xf trove-2.1.0.jar gnu
jar cfm mira.jar manifest.txt edu/ LICENSE LICENSE.trove gnu/
rm -rf gnu/
```
  1. Copy the jar to an executable location:
```
mkdir ~/bin
cp mira mira.jar ~/bin
```
  1. Add the executable location to your path variable (assumes bash):
```
export PATH=$PATH:~/bin
```
  1. Run mira:
```
mira
```
  1. Read [the](ChunkerTutorial.md) [tutorials](PosTaggerTutorial.md).