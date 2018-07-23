---
title: Course Archiving
category: System Administrator
---

After the term is completed you can archive the course.  This is done
per course, so it is easy to keep one or more courses 'open' to
allow a student to finish an incomplete, etc.  _Note: This procedure is
still a work in progress._

### Edit the Master Submitty Database Status

1. Switch to the `postgres` user:

    ```
    sudo su postgres
    ```

    Launch postgres:

    ```
    psql
    ```

    And connect to the master Submitty database:

    ```
    \c submitty
    ```


2. View all of the courses and their current status:

   ```
   select * from courses;
   ```

   or:

   ```
   select * from courses where semester='s18';
   ```

   **status = 1**
   The course is active.  All student users
   assigned to a non-NULL registration section, and all limited access
   graders, full access graders, and instructors can view the course.

   **status = 2**
   The course is archived.  Only instructor users for
   that course can view the course.  _TODO: We intend for this access
   to be read only, but that is currently not implemented._

   Other status codes may be defined in the future.


3. Change the status values as desired, for example:

   ```
   update courses set status=2 where semester='s18' and course='csci1100';
   ```

   or archive all courses for the semester:

   ```
   update courses set status=2 where semester='s18';
   ```


4. Exit postgres

   ```
   \q
   ```


### Convert your Course Files & Directories to _Read Only_

To prevent accidental modification, we recommend that you remove write
access from folders and files for archived courses.  And we suggest
you consider switching the group for your course to limit access to
the files of past semesters to the current instructors only.

```
chmod ugo-w /var/local/submitty/courses/<SEMESTER>/<COURSE>
```

Note that both of these changes can be nontrivial to revert (since the
permissions and group ownership are nonuniform within the course
directory.

_TODO: Add more instructions and notes as the procedure is developed._


### Backup your Data

Now's also a good time to make a final backup the data for the course.

```
/var/local/submitty/courses/<SEMESTER>/<COURSE>
```

We recommend that you preserve the submission files for the course in
the current location if you plan to run Lichen Plagiarism Detection
against past terms.

```
/var/local/submitty/courses/<SEMESTER>/<COURSE>/submissions
```


You may also want to backup and/or delete the relevant version control system (VCS)
repositories from:

```
/var/local/submitty/vcs/
```

Note that a checkout of the files from the repo is stored in this
folder for each submission click.  _Note: We intend to revise
autograding to perform a shallow clone of the repository so this will
not be a full clone with complete repository history._

```
/var/local/submitty/courses/<SEMESTER>/<COURSE>/checkout
```

Finally you may want to make a dump of the current contents of the database.

_TODO: Add instructions... _

### Archive Database Data

#### Dump Course Database

1. Switch to `postgres` user:

    ```
    sudo su postgres
    ```

2. Use `pg_dump` to dump your course database with data.  We are writing to `/tmp` to ensure that `postgres` has write permissions.

    ```
    pg_dump --create --clean --if-exists submitty_s18_csci1100 > /tmp/s18_csci1100.dbdump
    ```

3. Change ownership of dump file and move it out of `/tmp`.

#### Dump Course Related Data From Master Database

1. Switch to `postgres` user (if not already `postgres` from previous section):

   ```
   sudo su postgres
   ```

   Launch postgres:

   ```
   psql
   ```

   And connect to the master Submitty database:

   ```
   \c submitty
   ```

2. Dump courses_users table.  We are writing to `/tmp` to ensure that `postgres` has write permissions.

   ```sql
   copy courses_users to '/tmp/s18_csci1100.courses_users' (select * from courses_users where semester='s18' and course='csci1100');
   ```

3. Dump registration_sections information.

   ```sql
   copy courses_registration_sections to '/tmp/s18_csci1100.registration_sections' (select * from courses_registration_sections where semester='s18' and course='csci1100');
   ```

4. Dump users table.

   ```sql
   copy users to 'tmp/s18_csci1100.users' (select u.* from users as u left outer join courses_users as cu on u.user_id=cu.user_id where cu.semester='s18' and cu.course='csci1100');
   ```

5. Change ownership of dump files and move them out of `/tmp`.

