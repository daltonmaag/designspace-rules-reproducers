# Reproducers for Designspace Rule Behavior

1. Compile the test and reference fonts with
   - `fontmake -m var-reference.designspace -o variable --output-dir out`
   - `fontmake -m var-test.designspace -o variable --output-dir out`
   - `fontmake -m instance-reference.designspace -o ttf -i --output-dir out`
   - `fontmake -m instance-test.designspace -o ttf -i --output-dir out`
2. In a tester like https://www.corvelsoftware.co.uk/crowbar/, use the test string "Aa".
3. For the variable fonts, set the stack axis to 1 and play with the weight axis slider.
4. Compare the test fonts to the reference font.

![How it should look](https://github.com/daltonmaag/designspace-rules-reproducers/assets/380829/e2c5956b-a0ef-4fa4-ba9d-ef6136c1e2b8 "Reference behavior")

## Designspaces Explained

Tested as of fontmake 3.9.0.

### var-test.designspace

Here, the rules are written like what would be most convenient for humans.

1. You start with a catch-all rule at Stack=1 that substitutes the normal `A` and `a` glyphs with their alternate variant
2. Subsequent rules should refine those replacements depending on the position of the height axis.

### var-reference.designspace

This has the desired behavior, written in a way that works.

1. All rules have been manually rearranged to not overlap (doing the work of `fontTools.varLib.featureVars.overlayFeatureVariations`).
2. As such, the lookups don't build on each other, but always start from the base `A` and `a` glyphs, so lookup order doesn't matter.

This is tedious and error-prone to do manually, and duplicates substitutions.

### instance-reference.designspace

This has the desired behavior, written in a way that works for instance generation.

Here the rules are again convenient for humans, but with a twist.

Instead of being able to use the same rules as in var-test.designspace, you have to change the substitutions to always start with the `A` and `a` glyphs, because the instantiator _swaps_ glyph contents using all rules that match in order. Chaining glyph substitutions like A → B and then in a later rule B → C need to be done like A → B and then A → C.

### instance-test.designspace

We can't use the rules in var-reference.designspace for instances.

The rule condition set start and end points overlap to avoid tiny gaps in the variable font. Because the instantiator goes over all rules that match a location, on rule boundaries (here: Height, HGHT=750), it is going to collect two rules (here: `bracket1` and `bracket2`). The rules contain duplicate substitutions, which causes the instantiator to swap the same glyphs twice in a row, undoing the substitution.

## Conclusion

For variable fonts, overlapping rules have to be cut up and rearranged to not overlap, because shapers will take the first condition set that matches and run all the lookups registered there and then stop. Rule lookups may not end up in the same order in the binary as in the Designspace, probably because of that, making it hard to have them build on each other.

For instance generation, the instantiator is going to run all rules whose condition set matches, in order, and is going to swap glyphs rather than chaining their substitution.

This behavioral difference means that if you can't use manually written variable feature files and instance cutting, you have to write two Designspaces, one with rules for variable fonts and one with rules for instance generation.
