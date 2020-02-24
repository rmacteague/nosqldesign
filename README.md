# Application Background

Multi-Tenant wellness application that allow admin users to create program plans for their employees.  Plans run over a set period of time, typically a calendar year and include different wellness engagements for their employees to participate in.  Key requirements of the application is the ability for admins to generate custom engagement vehicles (such as a walking activity/challenge) to drive wellness imporvement.  Global templates for these "vehicles" are provided, admins can also upload custom content and create their own templates.  

# Activity Micro-Service

This portion of the application is responsible for activity generation and management.  There are 2 main activity types (events and challenges) each with 5 - 10 sub types.  For example an activity can be fitness based (walking challenge) or education based (take 3 courses on effects of alcohol use).  

One odd caveat is activities can be grouped in sections.  Didnt want to make SectionId part of the partition hiearchy in this case as admins move activities to and from sections routinely thus forcing a delete/insert in partition hiearchy.  A sectionId pointer is included as attribute on Activity to group when necessary.  We group in the application code but they both return in same query using Pattern 1 below.  

# Core Access Patters
We have over 50 access patterns but I will keep it to 5 simple ones for simplicity/brevity sakes.  

# Pattern 1
Get all active activities and sections within current Program plan year (01/01/2020 - 12/31/2020)

Keys: 
GSI1PK(CompanyId#Program)  GSI1SK(Where Activity End Date Between (01/01/2020 - 12/31/2020)

Projection: 
Many projected non-key attributes including a Data column for grouping non record of truth metadata (this helped staying under 20 projection limit)

ResultSetExample(GSI1PK, GSI1SK, PK, Name, Data): 
CompanyId-123#Program | EndDate#2020-12-31T00:00:00 | ActivityId-123 | Walk 5000 | Data: {grouped no record of truth data}

# Pattern 2:
Get list of active activities in program year.  In many parts of application we need to create either simple grids, drop downs, dashboards for applications or to find activities for back end processing.  A simple example of this is whenever a new employee starts a lambda is triggered to add them to eligible challenges.  There is a filter map on the base program row so we want that included in our projection (there are too many filter options to index each one so we are fine with pulling full set back and filtering in memory.  Companies usually max out at 50-100 activites per year.  

Keys: 
GSI2PK(CompanyId#Program#List)  GSI2SK(Where Activity End Date Between (01/01/2020 - 12/31/2020)

Projection: Very narrow projection here as index is used for lists (Name, Filters, PK, Type, SubType, Status)

ResultSetExample(GSI1PK, GSI1SK, PK, Name, Data): 
CompanyId-123#Program | EndDate#2020-12-31T00:00:00 | ActivityId-123 | Walk 5000 | { Filters: { Type: Challenge, SubType: Fitness, Roles: ["EmployeeOnPlan"], Depts: ["Sales"]}}

ApplicationFilter: 
We can either use FilterExpressions or Application Business Logic to further filter down data set based on user criteria

# Pattern 3:
Get list of user activity registrations for program year. Example usage we pair this with Pattern 1 to determine what to show on home page for user for current, active and completed activities within a section.    

Keys: 
GSI3PK(UserId#Registrations)  GS31SK(Where Activity End Date Between (01/01/2020 - 12/31/2020)

Projection: 
Many projected non-key attributes including a Data column for grouping non record of truth metadata (this helped staying under 20 projection limit)

ResultSetExample(GSI1PK, GSI1SK, PRG_Id, Progress, ProgressDisplayTest, EarnedPoints):
UserId-123#Registrations | ActivityEndDate#2020-12-31T00:00:00 | ActivityId-789 | 40.4 | 404 / 1000 steps | 40

# Pattern 4:
Get all information about a program.  Example usage (Detail Screen, Reward Processing, Reports)

Keys: 
PK(ActivityId) SK(Hiearchy of how much detail we would like)

ResultSetExamples (Full Detail) PK: Eqauls(ActivityId-123 | SK: BeginsWith(Program)
ActivityId-123 | Program
ActivityId-123 | Program#Detail (Desc, Other Non Core Data)
ActivityId-123 | Program#GoalPrimary (Type, Parameters)
ActivityId-123 | Program#Reward#Attendance (Name, Amount)
ActivityId-123 | Program#Groups (Group Type / Rules)
ActivityId-123 | Program#Groups#GroupId-123 (Name, Progress)
ActivityId-123 | Program#Groups#GroupId-456 (Name, Progress)

Limited Hiearchy Query (Admin Screen List of Groups For Activty-123) PK: Eqauls(ActivityId-123 | SK: BeginsWith(Program#Groups)
ActivityId-123 | Program#Groups (Group Type / Rules)
ActivityId-123 | Program#Groups#GroupId-123 (Name, Progress)
ActivityId-123 | Program#Groups#GroupId-456 (Name, Progress)

# Pattern 5:
Below is a pattern used repeatedly for activities for many different facets that have a many to many relationship (venues, files, courses, trackers, notifications).  This is potentially controversial and I'm curious on feedback if this is correct pattern to use.  

Example:
Companies are able to create many venues to be used within many activities of type event.  For example we have a quarterly blood drive that takes place at Blood Bank A at 211 Wabash Street.  To optimize the read when a location is selected for an event I copy the pertinent data (name, address details) into the data column of the Activty#Venue recordd Data attribute.  Two weeks into year find out Blood Bank A is closing.  

AccessPattern:
I need to find all activities ids using VenueId-123(Blood Bank A) and update to use VenueId-456(Blood Bank B).

Keys: 
GSI2PK(VenueId) GSI2SK(Status#Date)

Projection: 
(ACT_Id, Data)

ResultSetExample(GSI1PK: Equals(VenueId-123) | GSI1SK: Between(Today && 2020-12-31T00:00:00): 
ActivityId-123 | Data: {Name: Blood Bank A, Addr1: 211 Wabash Street, City: Chicago, ST: IL, Postal: 60606}
ActivityId-456 | Data: {Name: Blood Bank A, Addr1: 211 Wabash Street, City: Chicago, ST: IL, Postal: 60606}
ActivityId-789 | Data: {Name: Blood Bank A, Addr1: 211 Wabash Street, City: Chicago, ST: IL, Postal: 60606}

With the result set I would BatchWrite deletes of VenueId-123 and puts of VenueId-456.  

Similarly if it was an update to Venue-123 address from 211 to 311 Wabash Street, same query would just run an update on each records Data attribute.  
