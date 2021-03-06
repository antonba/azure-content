### Create a console application

- Launch Visual Studio and create a new Console application.

### Add the Relay NuGet package

1. Right-click the newly created project and select **Manage NuGet Packages**.

2. Click the **Browse** tab, then search for "Microsoft Azure Relay" and select the **Microsoft Azure Relay** item. Click **Install** to complete the installation, then close this dialog box.

### Write some code to send messages

1. Add the following `using` statement to the top of the Program.cs file.

    ```cs
    using Microsoft.Azure.Relay;
    ```

2. Add constants to the `Program` class for the Hybrid Connection connection details. Replace the placeholders in brackets with the proper values that were obtained when creating the Hybrid Connection.

    ```cs
    private const string RelayNamespace = "{RelayNamespace}";
    private const string ConnectionName = "{HybridConnectionName}";
    private const string KeyName = "{SASKeyName}";
    private const string Key = "{SASKey}";
    ```

3. Add a new method to the `Program` class like the following:

    ```cs
    private static async Task RunAsync()
    {
        Console.WriteLine("Enter lines of text to send to the server with ENTER");

        // Create a new hybrid connection client
        var tokenProvider = TokenProvider.CreateSharedAccessSignatureTokenProvider(KeyName, Key);
        var client = new HybridConnectionClient(new Uri(String.Format("sb://{0}/{1}", RelayNamespace, ConnectionName)), tokenProvider);

        // Initiate the connection
        var relayConnection = await client.CreateConnectionAsync();

        // We run two conucrrent loops on the connection. One 
        // reads input from the console and writes it to the connection 
        // with a stream writer. The other reads lines of input from the 
        // connection with a stream reader and writes them to the console. 
        // Entering a blank line will shut down the write task after 
        // sending it to the server. The server will then cleanly shut down
        // the connection which will terminate the read task.

        var reads = Task.Run(async () => {
            // initialize the stream reader over the connection
            var reader = new StreamReader(relayConnection);
            var writer = Console.Out;
            do
            {
                // read a full line of UTF-8 text up to newline
                string line = await reader.ReadLineAsync();
                // if the string is empty or null, we are done.
                if (String.IsNullOrEmpty(line))
                    break;
                // write to the console
                await writer.WriteLineAsync(line);
            }
            while (true);
        });

        // read from the console and write to the hybrid connection
        var writes = Task.Run(async () => {
            var reader = Console.In;
            var writer = new StreamWriter(relayConnection) { AutoFlush = true };
            do
            {
                // read a line form the console
                string line = await reader.ReadLineAsync();
                // write the line out, also when it's empty
                await writer.WriteLineAsync(line);
                // quit when the line was empty
                if (String.IsNullOrEmpty(line))
                    break;
            }
            while (true);
        });

        // wait for both tasks to complete
        await Task.WhenAll(reads, writes);
        await relayConnection.CloseAsync(CancellationToken.None);
    }
    ```

4. Add the following line of code to the `Main` method in the `Program` class.

    ```cs
    RunAsync().GetAwaiter().GetResult();
    ```

    Here is what your Program.cs should look like.

    ```cs
    using System;
    using System.IO;
    using System.Threading;
    using System.Threading.Tasks;
    using Microsoft.Azure.Relay;

    namespace Client
    {
        class Program
        {
            private const string RelayNamespace = "{RelayNamespace}";
            private const string ConnectionName = "{HybridConnectionName}";
            private const string KeyName = "{SASKeyName}";
            private const string Key = "{SASKey}";

            static void Main(string[] args)
            {
                RunAsync().GetAwaiter().GetResult();
            }

            private static async Task RunAsync()
            {
                Console.WriteLine("Enter lines of text to send to the server with ENTER");

                // Create a new hybrid connection client
                var tokenProvider = TokenProvider.CreateSharedAccessSignatureTokenProvider(KeyName, Key);
                var client = new HybridConnectionClient(new Uri(String.Format("sb://{0}/{1}", RelayNamespace, ConnectionName)), tokenProvider);

                // Initiate the connection
                var relayConnection = await client.CreateConnectionAsync();

                // We run two conucrrent loops on the connection. One 
                // reads input from the console and writes it to the connection 
                // with a stream writer. The other reads lines of input from the 
                // connection with a stream reader and writes them to the console. 
                // Entering a blank line will shut down the write task after 
                // sending it to the server. The server will then cleanly shut down
                // the connection which will terminate the read task.

                var reads = Task.Run(async () => {
                    // initialize the stream reader over the connection
                    var reader = new StreamReader(relayConnection);
                    var writer = Console.Out;
                    do
                    {
                        // read a full line of UTF-8 text up to newline
                        string line = await reader.ReadLineAsync();
                        // if the string is empty or null, we are done.
                        if (String.IsNullOrEmpty(line))
                            break;
                        // write to the console
                        await writer.WriteLineAsync(line);
                    }
                    while (true);
                });

                // read from the console and write to the hybrid connection
                var writes = Task.Run(async () => {
                    var reader = Console.In;
                    var writer = new StreamWriter(relayConnection) { AutoFlush = true };
                    do
                    {
                        // read a line form the console
                        string line = await reader.ReadLineAsync();
                        // write the line out, also when it's empty
                        await writer.WriteLineAsync(line);
                        // quit when the line was empty
                        if (String.IsNullOrEmpty(line))
                            break;
                    }
                    while (true);
                });

                // wait for both tasks to complete
                await Task.WhenAll(reads, writes);
                await relayConnection.CloseAsync(CancellationToken.None);
            }
        }
    }
    ```