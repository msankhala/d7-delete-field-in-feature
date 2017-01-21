# Deleting a Drupal Field Saved in Features

There comes a time in every Drupal developer's day when they realize they really don't need a field on some content type or other entity. Maybe it was a field holding temporary data while you did a migration or maybe a feature was removed or maybe things have been rearchitected to use a non-field approach.

Deleting a field might not sound terribly difficult but it does require being very careful and meticulous... particularly if that field has made it into production. With the help of drush, things get easier but, if you are using field groups and/or Display Suite, you might have extra steps that aren't solved by drush (unless you happen to know how to do this via drush... if so, leave a comment ;).

Assumptions
I will assume:

- you are deleting a field from a content type
- the field is live on the production site
- you have a local => dev => stage => live development process
- you are using git
- the field configuration is saved in a feature (via Features module)
- Let's say the field is called field_marketing_blurb, the content type is called marketing_post, the feature module is called marketing and the git branch we're working is called deleteblurb.

## Deleting the field locally

1) Get your code/branch sync'ed up with master


2) Use a recent copy of the live database


3) Revert all your features to make sure you are in sync:
```
drush fra --yes
```

4) Make sure things really did revert successfully (you have no overrides)
```
drush features-list
```

5) Check your code to see where the field is being used:
```
grep -r field_marketing_blurb .
```

6) If you have custom code that refers to these fields, you'll need to update the code to pull these out


7) If there are views, rules, context, etc. referring to the field, you'll need to update these to pull these out (you will know if they are used if you have * been saving your config into features code ;)


8) Check again to make sure you removed references beyond the content type (step #5) (the content type field info will refer to base and instance configuration)


9) Now you can delete the field with drush:
```
drush field-delete field_marketing_blurb
```
NOTE: You can instead add a hook_update_N function to a custom module and * delete the field with code (see field_delete_field).


10) Check the field was actually deleted:
```
drush field-info fields | grep field_marketing_blurb
```

11) To be really sure there is nothing left in database, you can do some queries:
```
show tables like '%field_marketing_blurb%'
select * from variable where name like '%field_marketing_blurb%';
```

12) Once you are all clear, then you can recreate your feature that included the field:
```
drush fu --yes marketing
```

13) Now there should be nothing in the codebase referring to this field:
```
grep -r field_marketing_blurb .
```

14) But, if you are using field_group, you might have some references left for that so go to your managed fields page for the content type and click the save  button and then repeat steps 12 and 13


15) And, if you use Display Suite, you might have some references left for that so go to the manage display page for *EACH* view mode and click save and then * repeat steps 12 and 13


16) If you still have some references in your feature module, you'll need to look at the code to see what is referring to it and handle accordingly (for most cases, the steps above will work)


17) VERY IMPORTANT: for each feature you have changed (and any custom code you * change), make sure that the code changed makes sense:
```
git diff marketing
```

18) Once you are convinced that all is well and you aren't introducing changes you shouldn't, commit the code:
```
git commit marketing
```

19) Now, we'll double check that features is happy by reverting the feature and recreating:
```
drush fr --yes marketing
drush fu --yes marketing
git diff marketing
```

20) You shouldn't see any changes

## Testing your changes

1) Test locally and make sure things are work and/or run automated tests if you have them

2) Push code to your branch:
```
git push origin deleteblurb
```

3) If you have peer review cycle, have someone sanity check your branch before merging:
```
git checkout deleteblurb
git diff origin/master
```

4) If all good, merge code and push:
```
git checkout master
git merge deleteblurb
git push origin master
```

5) On dev site, revert all features and clear cache (this assumes dev site uses master branch code):
```
drush @alias-to-dev-site fra --yes
drush @alias-to-dev-site cc all
```

6) On dev site, run drush field-delete:
```
drush @alias-to-dev-site field-delete field_marketing_blurb
```

7) NOTE: If using hook_update_N and field_delete_field then you just need to run drush @alias-to-dev-site updatedb.

8) Test on dev site

9) If all good, merge code to test branch (or, depending on your workflow, tag and push to test)


10) On test site, revert all features and clear cache:
```
drush @alias-to-test-site fra --yes
drush @alias-to-test-site cc all
```

11) On test site, run drush field-delete:
```
drush @alias-to-test-site field-delete field_marketing_blurb
```

12) NOTE: If using hook_update_N and field_delete_field then you just need to run 
```
drush @alias-to-test-site updatedb.
```

13) Test on test site

14) Have client test on test site

## Push changes to live

1) Backup database

2) Put site into maintenance mode

3) Push code to live

4) Revert all features and clear cache:
```
drush @alias-to-live-site fra --yes
drush @alias-to-live-site cc all
```

5) Make sure features are reverted
```
drush @alias-to-live-site features-list
```

6) CAREFULLY delete field via drush (have command handy for easy copy/pasting to minimize error):
```
drush @alias-to-live-site field-delete field_marketing_blurb
```

7) NOTE: If using hook_update_N and field_delete_field then you just need to run 
```
drush @alias-to-live-site updatedb.
```

8) Make sure field is deleted
```
drush @alias-to-live-site field-info fields | grep field_marketing_blurb
```

9) Test site

10) Disable maintenance mode
