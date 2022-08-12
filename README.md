#Solution Summary
#Bugs Resolution
###Use Try-with-resources or close PreparedStatement in finally()
This is an issue because once PreparedStatement is opened, it remains open until we close it programmatically and this can cause memory leaks.
The solution is to use a try-with-resources block.
e.g.:
Code with issue - AllStudent.java
```
try (Connection Con = DB.getConnection()) {
PreparedStatement ps = Con.prepareStatement("select * from Users where Email like ?", ResultSet.TYPE_SCROLL_SENSITIVE, ResultSet.CONCUR_UPDATABLE);
ps.setString(1, Search);
ResultSet rs = ps.executeQuery();
ResultSetMetaData rsmd = rs.getMetaData();

            // remaining code //

                Con.close();
            } catch (Exception e) {
                System.out.println(e);
            }
```
Code after using try-with-resources:
```
try (Connection Con = DB.getConnection()) {
                try(PreparedStatement ps = Con.prepareStatement("select * from Users where Email like ?", ResultSet.TYPE_SCROLL_SENSITIVE, ResultSet.CONCUR_UPDATABLE);) {
                    ps.setString(1, Search);
                    ResultSet rs = ps.executeQuery();
                    ResultSetMetaData rsmd = rs.getMetaData();
                    
                    // remaining code //
                     
                    }
                }
                Con.close();
            } catch (Exception e) {
                System.out.println(e);
            }
```
We define the PreparedStatement as an argument in the try() clause (this is called try-with-resources) and the code that uses the PreparedStatement and it's ResultSet is wrapped with this try block. This will ensure that the PreparedStatement is exited automatically once the program exits the try block.
This same solution has been implemented for all cases where we have this bug.

###String and other boxed objects must use equals()
This bug occurs when we try to compare to Strings using the == operator. This will not work since in Java, Strings are defined as objects and not simply words. When we use == sign for comparison, it is actually comparing the location in memory of those strings. So, even if the strings contain the same words, they will probably not be equal since the memory location will be different.
The solution is to use .equals() method. E.g.
```
String a = "one";
String b = "one";
if (a==b) {
 //incorrect method
}
if (a.equals(b)) {
//correct method
}
```
###PreparedStatement has only 1 parameter
This happens when we define a PreparedStatement that requires only one parameter but provide arguments for more than one parameter.
e.g:
```
PreparedStatement ps = con.prepareStatement("SELECT * FROM Users where UserName = ?")
//The question mark represents a parameter. Here there is only one parameter, so only one argument is required.

ps.SetString(2,username); //This would be incorrect, since there is only one parameter, but we are trying to pass a second argument by specifying 2.
ps.SetString(1,username); //This is the correct way
```

##Password Masking
I have setup password masking by modifying the SQL queries that return the data to the screen - 
Original query: `select * from Users`
Modified query: `select UserID, '********' as UserPass,RegDate,UserName,Email from Users`
What I have done here is specifying the columns and substituting '********' for the UserPass field.
Same solution has been implemented for all cases.

##Validating EMail
I have added a new function that will check whether the e-mail ID already exists in DB in the class UserDao.java:
```
public static boolean CheckIfAlreadyEmail(String email) {
        boolean status = false;
        try {
            Connection con = DB.getConnection();
            String select = "select * from Users where Email=?";

            try(PreparedStatement ps = con.prepareStatement(select);) {
                ps.setString(1,email);
                ResultSet rs = ps.executeQuery();
                status = rs.next();
            }
            con.close();
        } catch (Exception e) {
            System.out.println(e);
        }
        return status;

    }
```
Then I'm calling this method as an additional condition in UserForm to validate before adding the User:
```
if (UsersDao.CheckIfAlready(User)) { //checks if username already exists
            JOptionPane.showMessageDialog(UserForm.this, "UserName already taken!", "Adding new User Error!", JOptionPane.ERROR_MESSAGE);
        } else if (UsersDao.CheckIfAlreadyEmail(UserEmail)) { //checks if email ID exists
            JOptionPane.showMessageDialog(UserForm.this, "Email already in use!", "Adding new User Error!", JOptionPane.ERROR_MESSAGE);
        } else {
            // Adds new user
```

## Preventing SQL Injection
Here SQL injection is possible because we're using Statements instead of PreparedStatements while validating the user. This means an attacker can pass some value like 1=1 to the input field which would return all data in the table.
This can be prevented by using PreparedStatements with parameters instead.
e.g. In LibrarianDAO validate method:
Vulnerable code:
```
  try {
            Connection con = DB.getConnection();
            String select = "select * from Librarian where UserName= '" + name + "' and Password='"+ password +"'";
            ResultSet rs = null;
            try(Statement selectStatement = con.createStatement();) {
                rs = selectStatement.executeQuery(select);
            }
            status = rs.next();
            con.close();
        } catch (Exception e) {
            System.out.println(e);
        }
```
Solution:
```
   try {
            Connection con = DB.getConnection();
            String select = "select * from Librarian where UserName=? and Password=?";
            try(PreparedStatement ps = con.prepareStatement(select);) {
                ps.setString(1,name);
                ps.setString(2,password);
                ResultSet rs = ps.executeQuery();
                status = rs.next();
            }
            con.close();
        } catch (Exception e) {
            System.out.println(e);
        }
```
By using PreparedStatement and replacing the direct injection of input into the query with parameters, the JDBC drivers will take care of validating the data before the query and can help prevent SQL injection.
Similar solution has been implemented in all cases where we are using Statement instead of PreparedStatement.