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

   ```sql
   select * from courses;
   ```

   or:

   ```sql
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

   ```sql
   update courses set status=2 where semester='s18' and course='csci1100';
   ```

   or archive all courses for the semester:

   ```sql
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
chmod -R ugo-w /var/local/submitty/courses/<SEMESTER>/<COURSE>
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

### Archive Database Data
These examples assume that the course is called "csci1100" and was taught in Spring 2018 ("s18").

#### Dump Course Database

1. Switch to `postgres` user:

   ```
   sudo su postgres
   ```

2. Use `pg_dump` to dump your course database with data.  We are writing to `/tmp` to ensure that `postgres` has write permissions.

   ```
   pg_dump --create --clean --if-exists submitty_s18_csci1100 > /tmp/s18_csci1100.dbdump
   ```

3. IMPORTANT: **Move** file out of `/tmp` to your course's data folder.
   - You may need to exit from the `postgres` user.

      ```
      exit
      ```

   - If you do not have a `db_backup` folder, create it.

      ```
      mkdir -p /var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups
      ```

   - Move `dbdump` file to the `db_backups` folder.  As the `dbdump` file is owned by `postgres` (needed to later restore this data), you'll need `root` to move it.

      ```
      sudo mv /tmp/s18_csci1100.dbdump /var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups
      ```

#### Dump Course Related Data From Master Database

1. Switch to `postgres` user (if not already `postgres` from previous section).

   ```
   sudo su postgres
   ```

   Launch Postgres and connect to master Submitty database.

   ```
   psql submitty
   ```

2. Dump `courses_users` data.  We are writing to `/tmp` to ensure that `postgres` has write permissions.

   ```sql
   copy (select * from courses_users where semester='s18' and course='csci1100') to '/tmp/s18_csci1100.courses_users';
   ```

3. Dump `courses_registration_sections` data.

   ```sql
   copy (select * from courses_registration_sections where semester='s18' and course='csci1100') to '/tmp/s18_csci1100.registration_sections';
   ```

4. Dump `users` data.

   ```sql
   copy (select u.* from users as u left outer join courses_users as cu on u.user_id=cu.user_id where cu.semester='s18' and cu.course='csci1100') to 'tmp/s18_csci1100.users';
   ```

5. Exit Postgres.

   ```
   \q
   ```

6. IMPORTANT: **Move** file out of `/tmp`.
   - You may need to exit from the `postgres` user.

      ```
      exit
      ```

   - If you do not have a `db_backup` folder, create it.

      ```
      mkdir -p /var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups
      ```

   - Move the files to the `db_backups` folder.  As they are owned by `postgres` (needed to later restore this data), you'll need `root` to move it.

      ```
      sudo mv /tmp/s18_csci1100.* /var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups
      ```

### Restore Course Database Data
These examples assume that the course is called "csci1100" and was taught in Spring 2018 ("s18").  The archive folder exists at `/var/local/submitty/archives`.

#### Restore Course Database

1. Switch to `postgres` user.

   ```
   sudo su postgres
   ```

2. Restore course database from dump.

   ```
   psql submitty_s18_csci1100 < /var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups/s18_csci1100.dbdump
   ```

#### Restore Course Data To Master Database

1. Switch to `postgres` user (if not already `postgres` from previous section).

   ```
   sudo su postgres
   ```

   Launch Postgres and connect to master Submitty database.

   ```
   psql submitty
   ```

2. Deactivate trigger functions.

   ```sql
   SET session_replication_role = replica;
   ```

3. Restore `users` table data.  The use of a temp table ensures that a user in multiple courses does not trigger a primary key violation.

   ```sql
   create temp table tmp_users (like users);
   copy tmp_users from '/var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups/s18_csci1100.users';
   insert into users (select * from tmp_users) on conflict do nothing;
   ```

4. Restore `courses_users` data.

   ```sql
   copy courses_users from '/var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups/s18_csci1100.courses_users';
   ```

5. Restore `courses_registration_sections` data.

   ```sql
   copy courses_registration_sections from '/var/local/submitty/courses/<SEMESTER>/<COURSE>/db_backups/s18_csci1100.registration_sections'
   ```
