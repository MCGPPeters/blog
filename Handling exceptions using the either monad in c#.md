
Eru.ExceptionHandling, a library for handling (transient) errors in .NET 
------------------------------------------------------------------------

Today just a short shout out for something I came up with during a code kata... I wondered how I would implement something like [Polly](https://github.com/michael-wolfenden/Polly) using purely (or at least as much as possible) functional concepts in C#. Some people would say I am suffering from the "not invented here syndrome"... but this is the way I learn... and when something (at least in my opinion) elegant comes out as a result, I will make it available. So after reading the post, do a "File->New->Project", install the package from NuGet called [Eru.ExceptionHandling](https://www.nuget.org/packages/Eru.ExceptionHandling/), give it try and let me know what you think by leaving comments and / or suggestions over here, on [GitHub](https://github.com/MCGPPeters/eru) or in the [Gitter](https://gitter.im/MCGPPeters/Eru?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) room...

On of the upsides of extremely pure functional languages, like Haskell, is that the signature of a function is always explicit about what it does. Side effects, meaning any additional action that is performed in a function other then contributing to calculating the return value, like exceptions and state full computations, are avoided or made explicit. As an example, consider the following C# function signature:

    public int Divide(int dividend, int divisor)
    {
		...
    }

Just from reading the signature, you wouldn't expect that this function might throw an exception, for instance when passing the value 0 (zero) to the parameter named 'divisor'. So how could we make that fact more explicit? Well, we could return that we either would return an integer when everything goes well or an exception indicating that something went wrong:
    
    public Either<Exception, int> Divide(int dividend, int divisor)
    {
    	...
    }

The Either type is used by functional programmers in exactly these kind of cases. The reason why the first type parameter is usually the the type of the failure condition is historical. Look at the type definition for clarification:

    public class Either<TLeft, TRight>
    {}

The type for TRight is used for the value when the outcome is "successful" (get it, Right?) so naturally TLeft is for the "failure".... So one could imagine something representing a "null" value as a TLeft, as it is the bane of our existence as C# programmers... among others...

Another practical example is connecting to a SQL database for instance, that could fail (intermittently)...

    public async Task<Either<Exception, SQLConnection>> Connect(string connectionString)
    {
    	...
    }

That's pretty explicit, isn't it? This function promises us to return either a SQLConnection that works or an Exception telling us what went wrong, sometime in the near future.

Now how can we implement this? Normally you would do something along the line of this I guess:

	public async Task<Either<Exception, SQLConnection>> Connect(string connectionString)
    {
     	try
		{
			return Task.FromResult(new Either(SqlConnection(connectionString)));
		}
		catch(Exception ex)
		{
			return Task.FromResult(new Either(ex));
		}
    }

Add some transient error handling with retry logic etc... and you can so it can get messy pretty fast, while the intention of your code won't get clearer. And how would you do this in a generic, repeatable way?

To make this all more expressive, repeatable, composable and concise, the results of my learning process can be used. 

Try once:

	public async Task<Either<Exception, SQLConnection>> Connect(string connectionString)
    {
     	return await new SqlConnection()
		    .Try(connection =>
		    {
		    	connection.Open();
		    	return connection;
		    });
    }

Retry until some condition is true (for instance a transient error doesn't occur again):


    public async Task<Either<Exception, SQLConnection>> Connect(string connectionString)
    {
		Func<bool> itsNotTheEndOfTime = () => true;

     	return await new SqlConnection()
		    .Retry(connection =>
		    {
		    	connection.Open();
		    	return connection;
		    })
		    .WhileAsync(itsNotTheEndOfTime);
    }

These operations can be composed as well... consider the following (fully illustrative) example (View the tests on GitHub for more examples):

    var actualResult = 
		await new SqlConnection()
		    .Retry(connection =>
            {
                connection.Open();
                return connection;
            })
            .While(somePredicate)
            .Try(connection =>
            {
                SqlCommand command = new SqlCommand("SELECT OrderID, CustomerID FROM dbo.Orders;", connection);
                command.ExecuteReader();
                return command;
            });	

Again, let me know what you think...  Thumbs up / down?