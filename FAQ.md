# Exceptions #
```
java.io.InvalidClassException: edu.lium.mira.Mira; local class incompatible: stream classdesc serialVersionUID = 3, local class serialVersionUID = 4
```
Means that your model version is deprecated. You can either convert it to text with the old mira and back to binary with the new, or you can retrain everything with the new.