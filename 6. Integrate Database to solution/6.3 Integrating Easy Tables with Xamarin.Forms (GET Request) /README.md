# 6.3 Integrating Easy Tables with Xamarin.Forms (GET Request) 

### 6.3.1 Referencing Azure Mobile Services
At the earlier sections, we would have already added it to our Nuget Packages. If not

- For Visual Studio: Right-click your project, click Manage NuGet Packages, search for the `Microsoft.Azure.Mobile.Client` package, then click Install.
- For Xamarin Studio: Right-click your project, click Add > Add NuGet Packages, search for the `Microsoft.Azure.Mobile.Client` package, and then click Add Package.

NOTE: Make sure to add it to all your solutions!

If we want to use this SDK we add the following using statement
```Csharp
using Microsoft.WindowsAzure.MobileServices;
``` 

### 6.3.2 Creating Model Classes
Lets now create model class `NotHotDogModel` to represent the tables in our database. 
So in `Moodify (Portable)`, create a folder named `DataModels` and then create a `NotHotDogModel.cs` file with,

NOTE: If your table in your backend is not called timeline, rename it or rename this class and file to match.

```Csharp
public class NotHotDogModel
{
    [JsonProperty(PropertyName = "Id")]
    public string ID { get; set; }

    [JsonProperty(PropertyName = "Longitude")]
    public float Longitude { get; set; }

    [JsonProperty(PropertyName = "Latitude")]
    public float Latitude { get; set; }
}
``` 

- `JsonPropertyAttribute` is used to define the PropertyName mapping between the client type and the table 
- Important that they match the field names that we got from our postman request (else it wont map properly)
- Our field names for our client types can then be renamed if we want (like the field `date`)
- All client types must contain a field member mapped to `Id` (default a string). The `Id` is required to perform CRUD operations and for offline sync (not discussed) 

### 6.3.3 Initalize the Azure Mobile Client
Lets now create a singleton class named `AzureManager` that will look after our interactions with our web server. Add this to the class
(NOTE: replace `MOBILE_APP_URL` with your server name, for this demo its "https://nothotdoginformation.azurewebsites.net/")


So in `Moodify (Portable)`, create a `AzureManager.cs` file with,

```Csharp
public class AzureManager
    {

        private static AzureManager instance;
        private MobileServiceClient client;

        private AzureManager()
        {
            this.client = new MobileServiceClient("MOBILE_APP_URL");
        }

        public MobileServiceClient AzureClient
        {
            get { return client; }
        }

        public static AzureManager AzureManagerInstance
        {
            get
            {
                if (instance == null) {
                    instance = new AzureManager();
                }

                return instance;
            }
        }
    }
``` 

Now if we want to access our `MobileServiceClient` in an activity we can add the following line,
```Csharp
    MobileServiceClient client = AzureManager.AzureManagerInstance.AzureClient;
``` 


### 6.3.4 Creating a table references
For this demo we will consider a database table a `table`, so all code that accesses (READ) or modifies (CREATE, UPDATE) the table calls functions on a `MobileServiceTable` object. 
These can be obtained by calling the `GetTable` on our `MobileServiceClient` object.

Lets add our `timelineTable` field to our `AzureManager` activity 
```Csharp
    private IMobileServiceTable<Timeline> notHotDogTable;
``` 

And then the following line at the end of our `private AzureManager()` function
```Csharp
    this.timelineTable = this.client.GetTable<NotHotDogModel>();
```

This grabs a reference to the data in our `NotHotDogModel` table in our backend and maps it to our client side model defined earlier.

We can then use this table to actually get data, get filtered data, get a timeline by id, create new timeline, edit timeline and much more.

### 6.3.5 Grabbing timeline data
To retrieve information about the table, we can invoke a `ToListAsync()` method call, this is asynchronous and allows us to do LINQ querys.

Lets create a `GetHotDogInformation` method in our `AzureManager.cs` file
```Csharp
    public async Task<List<Timeline>> GetHotDogInformation() {
        return await this.notHotDogTable.ToListAsync();
    }
``` 

Lets create a button in our `HomePage.xaml` file after our other button
```xml
      <Button Text="See Timeline" TextColor="White" BackgroundColor="Red" Clicked="ViewTimeline_Clicked" />
``` 

Before the closing tag of the stacklayout in our `HomePage.xaml` file (`</StackLayout>`), add the following list view
```xml
  <?xml version="1.0" encoding="UTF-8"?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms" xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml" x:Class="Tabs.AzureTable" Title="Information">
    <ContentPage.Padding>
        <OnPlatform x:TypeArguments="Thickness" iOS="0, 20, 0, 0" />
    </ContentPage.Padding>
    <ContentPage.Content>
        <StackLayout>
            <Button Text="See Photo Information" TextColor="White" BackgroundColor="Red" Clicked="Handle_ClickedAsync" />
            <ListView x:Name="HotDogList" HasUnevenRows="True">
                <ListView.ItemTemplate>
                    <DataTemplate>
                        <ViewCell>
                            <StackLayout Orientation="Horizontal">
                                <Label Text="{Binding Longitude, StringFormat='Longitude: {0:N}'}" HorizontalOptions="FillAndExpand" Margin="20,0,0,0" VerticalTextAlignment="Center" />
                                <Label Text="{Binding Latitude, StringFormat='Latitude: {0:N}'}" VerticalTextAlignment="Center" Margin="0,0,20,0" />
                            </StackLayout>
                        </ViewCell>
                    </DataTemplate>
                </ListView.ItemTemplate>
            </ListView>
        </StackLayout>
    </ContentPage.Content>
</ContentPage>
```
Here we added a template for the timeline object values, showing the `Date`, `Happiness` and `Anger` values by using `Binding` ie `Text="{Binding Happiness}"`. This is a very simple way to display all our values and can be futher extended to display it in a aesthetic manner.
This associates the value of the field of the timeline object and displays it.


Now to can call our `GetTimelines` function, we can add the following method in our `HomePage.xaml.cs` class
```Csharp
    
        private async void ViewTimeline_Clicked(Object sender)
        {
            List<Timeline> timelines = await AzureManager.AzureManagerInstance.GetTimelines();
            
            TimelineList.ItemsSource = timelines;

        }
``` 

This will then set the source of the list view  `TimelineList` to the list of timelines we got from our backend

[More Info on ListView](https://developer.xamarin.com/guides/xamarin-forms/user-interface/listview/) about customising the appearance of your list view

[MORE INFO] A LINQ query we may want to achieve is if we want to filter the data to only return high happiness songs. 
We could do this by the following line, this grabs the timelines if it has a happiness of 0.5 or higher
```Csharp
    public async Task<List<Timeline>> GetHappyTimelines() {
        return await timelineTable.Where(timeline => timeline.Happiness > 0.5).ToListAsync();
    }
``` 

### 6.3.6 Posting timeline data
To post a new timeline entry to our backend, we can invoke a `InsertAsync(timeline)` method call, where `timeline` is a Timeline object.

Lets create a `AddTimeline` method in our `AzureManager.cs` file

```Csharp
    public async Task AddTimeline(Timeline timeline) {
        await this.timelineTable.InsertAsync(timeline);
    }
``` 

NOTE: If a unique `Id` is not included in the `timeline` object when we insert it, the server generates one for us.


Now to can call our `AddTimeline` function, we can do the following in our `HomePageXaml.cs` class at the end of the `TakePicture_Clicked` method so that each response from cognitive services is uploaded

Add this code after the  line `EmotionView.ItemsSource = result[0].Scores.ToRankedList();`

```Csharp
    var temp = result[0].Scores;

    Timeline emo = new Timeline()
    {
        Anger = temp.Anger,
        Contempt = temp.Contempt,
        Disgust = temp.Disgust,
        Fear = temp.Fear,
        Happiness = temp.Happiness,
        Neutral = temp.Neutral,
        Sadness = temp.Sadness,
        Surprise = temp.Surprise,
        Date = DateTime.Now
    };

    await AzureManager.AzureManagerInstance.AddTimeline(emo);
``` 

This creates a `Timeline` object and sets up the values from the `result` (from cognitive services) and then adds it to backends database

### 6.3.6 [More Info] Updating and deleting timeline data
To edit an existing timeline entry in our backend, we can invoke a `UpdateAsync(timeline)` method call, where `timeline` is a Timeline object. 

The `Id` of the timeline object needs to match the one we want to edit as the backend uses the `id` field to identify which row to update. This applies to delete as well.

Timeline entries that we retrieve by `ToListAsync()`, will have all the object's corresponding `Id` attached and the object returned by the result of `InsertAsync()` will also have its `Id` attached.

Lets create a `UpdateTimeline` method in our `AzureManager` activity 
```Csharp
    public async Task UpdateTimeline(Timeline timeline) {
        await this.timelineTable.UpdateAsync(timeline);
    }
``` 

NOTE: If no `Id` is present, an `ArgumentException` is raised.


To edit an existing timeline entry in our backend, we can invoke a `DeleteAsync(timeline)` method call, where `timeline` is a Timeline object. 
Likewise information concerning `Id` applies to delete as well.

Lets create a `DeleteTimeline` method in our `AzureManager` activity 
```Csharp
    public async Task DeleteTimeline(Timeline timeline) {
        await this.timelineTable.DeleteAsync(timeline);
    }
``` 

### Extra Learning Resources
* [Using App Service with Xamarin by Microsoft](https://azure.microsoft.com/en-us/documentation/articles/app-service-mobile-dotnet-how-to-use-client-library/)
* [Using App Service with Xamarin by Xamarin - Outdated but good to understand](https://blog.xamarin.com/getting-started-azure-mobile-apps-easy-tables/)
* [ListView in Xamarin](https://developer.xamarin.com/guides/xamarin-forms/user-interface/listview/)