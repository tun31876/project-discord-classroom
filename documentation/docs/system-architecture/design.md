---
sidebar_position: 1
---

# Architecture

## Class Diagram

```mermaid
---
title: Class Diagram
---
classDiagram
    bot <-- main
    FastAPI <-- bot
    supabase <-- bot
    discord <-- bot
    class main{
        +run_bot()
    }
    class bot{
        -configData: dictionary
        -DISCORD_TOKEN: string
        -PREFIX: char
        -SB_URL: string
        -SB_KEY: string
        -supabase: Client
        -bot: Bot
        -app: App
        +run_discord_bot()
        -on_ready()
        -on_guild_join(guide: string)
        -syllabus(ctx: context)
        -poll(ctx: context, arg1: string, arg2: string)
    }
    class FastAPI{
        +FastAPI(): App
    }
    class supabase{
        +create_client(): Client
    }
    class discord{
        +commands.Bot(): Bot
    }
```
The class diagram is made up of two python files, main.py and bot.py. Main's only purpose is to run bot.py. bot.py uses three seperate, non-native libraries: supabase, discord, and fastapi. The supabase library is used to connect with the database that is on supabase. It is connected through a URL and KEY pair and creates a Client object when connected. The discord library is used to connect with the Discord Bot through a Discord Token. Also, when creating the bot it needs to know the PREFIX for the commands which is "!" in our case. Finally, FastAPI is used to simplify our API calls to supabase. First we create an App object then give that object methods that are used for the API. 

## Database Design

```mermaid
---
title: Database Design
---
erDiagram
    CLASSROOM {
        int id PK
        string serverId
        string name
        int attendance
    }
    CLASSROOM_USER {
        int id PK
        int classroomId FK
        int userId FK
        string role
        int attendance
    }
    USER {
        int id PK
        string name
        string discordId
    }
    ASSIGNMENT {
        int id PK
        string name
        int points
        dateFormat startDate
        dateFormat dueDate
        int channelId
        int classroomId FK
    }
    QUIZ {
        int id PK
        string title
        int points
        dateFormat startDate
        dateFormat dueDate
        int timeLimit
        string questionsUrl
        int channelId
        int classroomId FK
    }
    DISCUSSION {
       int id PK
       int title
       int points
       dateFormat startDate
       dateFormat dueDate 
       int channelId
       int classroomId FK
    }
    GRADE {
        int id PK
        string taskType
        int taskId FK
        int studentId FK
        int graderId FK
        int score
    }
    CLASSROOM ||--|{ CLASSROOM_USER : has
    CLASSROOM_USER }|--|| USER : has
    CLASSROOM }|--|| ASSIGNMENT : is
    CLASSROOM }|--|| QUIZ : is
    CLASSROOM }|--|| DISCUSSION : is
    USER ||--o{ GRADE : has
```

Each time the bot is added to a Discord server a new row is added to the CLASSROOM table. This table holds discord server name and the total attendance and grade used to calculate student's grades and attendance scores. Each CLASSROOM contains one or more EDUCATORS and one or more STUDENTS. The STUDENT table holds the student's username, the classroom they belong to, their grade, and their attendance score. Their total grade will equal their grade divided by the CLASSROOM totalGrade. Next we have the ASSIGNMENT, QUIZ, and DISCUSSION tables. The ASSIGNMENT table keeps track of the assignments the EDUCATOR creates which includes the name of the assignment, when to make it available, and when its due. The QUIZ table keeps track of EDUCATOR created quizzes which holds the max score of the quiz, the start/due date, and an optional time limit for the quiz. Each QUIZ is made up of QUESTIONS which contain a prompt, a correct answer, and optional wrong answers depending on the type of question. (If no wrong answers then its a open-ended question or fill-in-the-blank, if one wrong answer could be a True/False, and if all wrong answers are given then its multiple choice). The DISCUSSION table is used to keep track of the Discussions within the Discord server. These will only include max scores and start/due dates. Finally the GRADES table holds all of the grades for the students.

## Sequence Diagrams

<details>
    
<summary>

### Use Case #1: Teacher !attendance command

```mermaid
sequenceDiagram
    actor Teacher
    actor Student1
    actor Student2
    participant Discord
    participant ClassroomBot
    participant Supabase DB
    Teacher->>Discord: User sends "!attendance" command
    activate Teacher
    activate Discord
    activate Student1
    activate Student2
    activate ClassroomBot

    Discord->>ClassroomBot: ClassroomBot reads command from Discord
    ClassroomBot->>Discord: message to react to for attendance
    Student1->>Discord: reacts to message
    Student2->>Discord: reacts to message
    Teacher->>Discord: command to close attendance
    Discord->>ClassroomBot: Attendance metrics
    ClassroomBot ->> Supabase DB: Record attendance for current message/session
    ClassroomBot ->> Discord: Session attendance summary
    Discord->> Teacher: Summary of the sessions attendance, + list of missing names
    deactivate Discord
    deactivate Teacher
    deactivate ClassroomBot
```

</summary>
<div>
<div>As a teacher user I want to record attendance of a lecture.</div>
<br/>

1. Teacher types `!attendance` command
2. The Bot reads the command and sends a attendance message to the discord
3. The students are able to react to the message
4. The teacher sends a command to close the attendance 
5. The bot checks the attendance metrics (by checking the reactions)
6. The bot sends the metrics to the Supabase Database
7. The bot sends the attendance summary to the teacher, with a list of missing students
    
</div>
    
</details>

<details>
    
<summary>

### Use Case #2: Student !grades command

```mermaid
sequenceDiagram
    actor Student
    participant Discord
    participant ClassroomBot
    participant FastAPI
    participant Supabase DB
    Student->>Discord: User sends "!grades" command
    activate Student
    activate Discord
    Discord->>ClassroomBot: ClassroomBot reads command from Discord
    activate ClassroomBot
    ClassroomBot->>FastAPI: GET grades from GRADES table where student_id == Student
    activate FastAPI
    FastAPI->>Supabase DB: API request to database
    activate Supabase DB
    Supabase DB-->>FastAPI: Returns list of GRADED assignments, quizzes, disucssions
    deactivate Supabase DB
    FastAPI -->> ClassroomBot: Sends list from Supabase to bot
    deactivate FastAPI
    ClassroomBot -->> Discord: DMs student their grades for each task
    deactivate ClassroomBot
    Discord-->> Student: Student reads DM from ClassroomBot
    deactivate Discord
    deactivate Student
```

</summary>
<div>
<div>As a student user I want to check my grades for the class.</div>
<br/>

1. The student types "!grades" command within the classroom discord server.
2. The ClassroomBot reads the command from the server
3. Using FastAPI an API GET request is made for the grades
4. The request is forwarded to the Supabase Database
5. Supabase returns the grades for that student as a list
6. FastAPI sends the list to the application
7. The application parses through the grades and neatly organizes them and direct messages the student their grades
8. The student reads their DMs to check their grades.
    
</div>
    
</details>

<details>
    
<summary>

### Use Case #3: Student takes practice quiz

```mermaid

sequenceDiagram
actor u as Student
participant d as Discord
participant c as ClassroomBot
participant f as FastAPI
participant s as Supabase DB

u->>d: Student types !pquiz in Quiz text channel
d->>c: Reads command from Discord
c->>f: GET list of current practice quizes from DataBase
f->>s: API request from DataBase
s-->>f: Return list of Practice Quizes
f-->>c: Sends list from DataBase to ClassRoom Bot
c-->>d: The Bot lists the available Practice Quizes
d-->>u: Student reads the list of Quizes they can take
u->>d: Student types !pquiz 2 in Quiz text channel
d->>c: Reads command from Discord
c->>f: GET Practice Quiz 2 from the DataBase
f->>s: API request from DataBase
s-->>f: Return Practice Quiz 2
f-->>c: Sends Practice Quiz 2 from the DatBase to ClassRoom Bot
c-->>d: The Bot Dms the student the questions for the practice quiz
d-->>u: Student reads the questions as they are messaged them
u->>d: Student answers the questions to the Bot via DM
d->>c: Reads answers and copies them
c->>f: PUSH answers to DataBase
f->>s: Record students answers
s-->>f: Return Student answers and Practice Quiz answers 
f-->>c: Compare the answers and Return Correct and incorect answers 
c-->>d: The Bot DMs the results to the student
d-->>u: Student knows where they stand on the topic by the results
```

</summary>
<div>
<div>This Diagram shows the process of a student wanting to take a Practice Quiz.</div>
<br/>

1. Student types !pquiz
2. The Bot reads the command and sends a request for the list of quizzes available to the API.
3. The API gets the data from the database and returns it to the Bot.
4. The Bot lists the available quizzes.
5. The Student reads the available quizzes and types !pquiz 2 to take the quiz they want.
6. The bot reads the command and send the request for the specific quiz to the API.
7. The API gets the questions from the database and returns them to the Bot.
8. The Bot DMs the student the questions.
9. The Student answers the questions.
10. The Bot reads the answers and pushes them to the API.
11. The API pushes the answers to the Database to be saved and then returns the answers key for the quiz and the student answers.
12. The API compares the two and returns the incorrect and correct answers to the Bot.
13. The Bot messages the Student their results.
14. The student knows where they stand on the topic due to their results.
    
</div>
    
</details>


<details>
    
<summary>

### Use Case #4: Student wants to ask the teacher a question

```mermaid

sequenceDiagram
    actor Student
    actor Teacher
    participant Discord
    participant ClassroomBot
    Student->>Discord: User sends "!ticketcreate" command
    activate Student
    activate Discord
    Discord->>ClassroomBot: ClassroomBot reads command from Discord
    activate ClassroomBot
    ClassroomBot->>Discord: creates a new private chat
    deactivate ClassroomBot
    activate Teacher
    Discord->>Teacher: Teacher is added to private chat
    Discord->>Student: Student is added to private chat
    Student->>Discord: Student asks question in chat
    Discord->>Teacher: Teacher receives question
    Teacher->>Discord: Teacher responds to question
    Discord->>Student: Student receives teacher's response
    deactivate Discord
    deactivate Teacher
    deactivate Student
    
```


</summary>
<div>
<div>This diagram shows a student asking a question to the teacher by creating a ticket for a private chat</div>
<br/>

1. Student types "!ticketCreate" command
2. ClassroomBot reads the command from discord
3. The bot creates a new private chat
4. The teacher and student are added to the private chat
5. Student can message the question to the teacher
6. Teacher responds to the students question
7. Student receives the teacher's response
    
</div>
    
</details>

<details>
    
<summary>

### Use Case #5: Educator creates poll with !poll

```mermaid

sequenceDiagram
    actor Teacher
    participant Discord
    participant ClassroomBot
    Teacher->>Discord: User sends "!pollcreate" command
    activate Discord
    Discord->>ClassroomBot: ClassroomBot reads command from Discord
    activate ClassroomBot
    ClassroomBot->>Discord: ClassroomBot prompts user for poll options
    Discord->>Teacher: Teacher receives poll prompt
    Teacher->>Discord: Teacher replies with poll question and options
    Discord->>ClassroomBot: ClassroomBot reads response from Discord
    ClassroomBot->>Discord: ClassroomBot publishes poll to Discord and sends confirmation to teacher
    Discord->>Teacher: Teacher receives confirmation that poll was created
    deactivate ClassroomBot
    deactivate Discord
    
```

</summary>
<div>
<div>This diagram shows an educator creating a poll for the students to respond to</div>
<br/>


1. The teacher enters the `!poll` command
2. The ClassroomBot reads the command from Discord
3. The bot prompts the user for the poll question and options
4. The teacher enters the specified information on Discord
5. The ClassroomBot reads the data from Discord
6. The ClassroomBot formats the poll message and publishes it to Discord
7. The teacher receives confirmation that their poll was created
    
</div>
    
</details>

<details>
    
<summary>

### Use Case #6: Educator takes attendance with !attendance command

```mermaid 

sequenceDiagram
    actor User
    participant Discord
    participant ClassroomBot
    participant Student1
    participant Student2
    participant FastAPI
    participant Database

    User->>Discord: !attendance
    activate Discord

    Discord->>ClassroomBot: attendance()
    deactivate Discord
    activate ClassroomBot

    ClassroomBot->>ClassroomBot: timerOn()
    
    loop
        ClassroomBot->> Student1: asks for an input (!present)
        activate Student1
        ClassroomBot->>Student1: remainingTime(5 minute)
        Student1-->>Discord: !present
        deactivate Student1
        Discord->>ClassroomBot: !present
        ClassroomBot->> Student2: asks for an input (!present)
        ClassroomBot->>Student2: remainingTime(5 minute)
    end

    Student2-->>Discord: null 
    Discord->>ClassroomBot: null
    ClassroomBot->>ClassroomBot: timerOff()
    ClassroomBot->>FastAPI: markAttendance()
    deactivate ClassroomBot
    activate FastAPI
    FastAPI->>Database:save(student1_attended)
    activate Database
    FastAPI->>Database:save(student2_absent)
    deactivate FastAPI
    deactivate Database    
    ClassroomBot->>FastAPI: retrieveAttendance()

    activate FastAPI
    FastAPI->>Database: retrieve_student_absent_list()
    
    activate Database

    Database-->>FastAPI: students_absent_list
    deactivate FastAPI
    deactivate Database
    activate FastAPI
    FastAPI-->>ClassroomBot: Students_absent_list
    deactivate FastAPI

    activate ClassroomBot
    ClassroomBot-->>Discord: Students_absent_list
    deactivate ClassroomBot
```

</summary>
<div>
<div>This diagram shows the process of recording students attendance. </div>
<br/>

1. User sends a command to the Discord server to initiate attendance tracking by typing `!attendance`.
2. The Discord server then passes the attendance request to the ClassroomBot.
3. The ClassroomBot starts a timer and begins a loop asking students in the classroom if they are present or not.
4. Student1  responds with a `"!present"` whereas Student2 does not.
5. Once loop is complete, the ClassroomBot deactivates and sends a request to FastAPI to mark the attendance.
6. The FastAPI stores the attendance data in a Database.
7. The FastAPI service then retrieves the list of students who were marked absent in the database and sends it back to the ClassroomBot.
8. The ClassroomBot activates again and receives the list of absent students from the FastAPI service.
9. The ClassroomBot sends the list of absent students to the Discord server for notification to the User.
    
</div>
    
</details>
