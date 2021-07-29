# ArtistTrack

## Introduction
Artist Track is an music application written in Python, using the Django framework. Users can enter the music they like into a database, and organize it into playlists. Users can edit and delete songs, and this application could be built using Spotify's API, so users can log in to their spotify account and play their music. A user of Artist Track can also get lyrics for any song on the internet, and artist information provided by scraping the artist's wikipedia page.  

### Stories
* [Create](#create)
* [Read](#read)
* [Update and Delete](#update-and-delete)
* [Web Scraping](#web-scraping)
* [API](#api)
* [Front End Development](#front-end-development)
* [Skills](#skills)

### Create
After creating the basic app, I was tasked with creating the database model. I chose a many to many relationship between my tables, because a playlist should be able to have many songs, and a song should be able to be on multiple playlists. After some feedback, it seemed better for the year field to have a drop down with choices, instead of having to enter it in manually, so I used list comprehension to create a list of years from 1950 to 2022, and I made it descending so that the more common recent years would be closer for selection.  

```
class Song(models.Model):
    GENRE_CHOICES = (
        ('rock', 'rock'),
        ('pop', 'pop'),
        ('classical', 'classical'),
        ('heavy metal', 'heavy metal'),
        ('other', 'other'),
    )

    YEAR_CHOICES = [(r, r) for r in range(2022, 1950, -1)]

    song_name = models.CharField(max_length=50)
    artist = models.CharField(max_length=50)
    album = models.CharField(max_length=50, default=None, blank=True)
    genre = models.CharField(max_length=50, choices=GENRE_CHOICES)
    year = models.IntegerField(choices=YEAR_CHOICES)
    lyrics = models.TextField(default=None, null=True)

    Songs = models.Manager()

    def __str__(self):
        return "{} by {}".format(self.song_name, self.artist)


class Playlist(models.Model):
    playlist_name = models.CharField(max_length=50)
    playlist_description = models.TextField()
    playlist_songs = models.ManyToManyField(Song)

    Playlists = models.Manager()

    def __str__(self):
        return self.playlist_name

```

### Read
Stories 3 and 4 involved displaying and editting the information entered into the database from the user. I created a library page that would display songs, organized by playlists, using Django Template Language and a nested for loop. I also created a page where all songs in the database are displayed in a table with columns for artist, title, album, genre, and year.

```
<div class="at-library-container">
{% for playlist in playlists %}
    <div class="at-opaque-container at-playlist-table ">
        <a href="{{ playlist.pk }}/playlist_details"><h4 class="red-text text-center">{{ playlist.playlist_name }}</h4></a>
        <p class="text-white text-center">{{ playlist.playlist_description }}</p>
        <table class="table table-bordered">
            <tr class="red-text">
                <th scope="col">Artist</th>
                <th scope="col">Song</th>
            </tr>
            <!--.all is what allows me to iterate through that list of songs and access the relevant DB information-->
            {% for song in playlist.playlist_songs.all %}
            <tr class="text-white">
                <td scope="row"><a href="{{ song.pk }}/artist_info">{{ song.artist }}</a></td>
                <td scope="row"><a href="{{ song.pk }}/details">{{ song.song_name }}</a></td>
                    {% endfor %}
            </tr>
        </table>
    </div>
{% endfor %}
</div>
{% endblock %}

```

### Update and Delete
Here I created a page for editting and deleting items from the database. For this I created Django forms for both tables. The widgets allow me to style the form. I also added delete functionality for both tables, including a confirm delete page. 


```
class SongForm(ModelForm):
    class Meta:
        model = Song
        fields = ('song_name', 'artist', 'album', 'genre', 'year')
        
        widgets = {
            'song_name': forms.TextInput(attrs={'class': 'form-control', 'placeholder': 'Song Name'}),
            'artist': forms.TextInput(attrs={'class': 'form-control', 'placeholder': 'Artist'}),
            'album': forms.TextInput(attrs={'class': 'form-control', 'placeholder': 'Album'}),
            'genre': forms.Select(attrs={'class': 'form-control'}),
            'year': forms.Select(attrs={'class': 'form-control', 'placeholder': 'Year'}),
        }


class PlaylistForm(ModelForm):
    class Meta:
        model = Playlist
        fields = '__all__'

        widgets = {
            'playlist_name': forms.TextInput(attrs={'class': 'form-control', 'placeholder': 'Playlist Name'}),
            'playlist_description': forms.TextInput(attrs={'class': 'form-control', 'placeholder': 'a description about your playlist'}),
            'playlist_songs': forms.SelectMultiple(attrs={'class': 'form-control'})
        }
        
```

### Web Scraping
Stories 6 and 7 involved web scraping. For this, I decided to use wikipedia, since they allow web scraping and have information on a lot of artists. I was able to set up a view function that takes the artist and title that the user selected from the database, adds it to the wikipedia base url and gets the page. From there, I used beautiful soup to find the first three paragraphs, which contain the general artist information, and extract the plain text out for display on the artist info page. At first, if the user clicked on the link to this page, and results were not found, nothing would happen, so I added timeout=20, and wrapped the request in a try/catch statement so if the request times out, or the information is not found, the user is routed to the artist info page and a message is displayed.

``` 

def at_artist_info(request, pk):
    pk = int(pk)
    song = get_object_or_404(Song, pk=pk)
    artist = song.artist
    while True:
        try:
            response = requests.get("https://en.wikipedia.org/wiki/{}".format(artist), timeout=20)
            soup = BeautifulSoup(response.content, features="html.parser")
            # gets the first three paragraphs of the artist page from wikipedia, which contain the general description.
            html_string = soup.find_all('p')[0]
            html_string2 = soup.find_all('p')[1]
            html_string3 = soup.find_all('p')[2]
            # returns the plain text of the html string
            band_info = html_string.text
            band_info2 = html_string2.text
            band_info3 = html_string3.text
            content = {
                'song': song,
                'band_info': band_info,
                'band_info2': band_info2,
                'band_info3': band_info3
            }
            return render(request, 'ArtistTrack_artistInfo.html', content)
        except:
            # send the message into the dictionary as band info
            band_info = 'Something went wrong. Most likely either the artist name was spelled wrong, ' \
                        'or wikipedia does not have a page for this artist.'
            content = {
                'song': song,
                'band_info': band_info
            }
            return render(request, 'ArtistTrack_artistInfo.html', content)

```

### API
I was tasked with using an API for this project. I chose lyrics.ovh, that gets the lyrics for any song on the internet. The function takes the artist and title of a song entered by the user into the database, and grabs the lyrics for that song, translates them into a readable format and prints them to the template. Later, I added some functionality, by saving the lyrics to the database, and wrapping the whole thing in a if statement, so if the user has already gotten the lyrics once, there is no need to make another API call, it simply prints the lyrics from the database. 

```

def at_lyrics_api(request, pk):
    pk = int(pk)
    song = get_object_or_404(Song, pk=pk)
    # api takes artist and title as parameters
    artist = song.artist
    title = song.song_name
    if song.lyrics is None:
        while True:
            try:
                # set timeout to 20, after this point, it's likely not going to find the song
                response = requests.get("https://api.lyrics.ovh/v1/{}/{}".format(artist, title), timeout=20)
                json_data = json.loads(response.content)
                lyrics = json_data['lyrics']
                song.lyrics = lyrics
                song.save()
                context = {
                    'song': song,
                    'lyrics': lyrics,
                }
                return render(request, "ArtistTrack_lyrics.html", context)
            # set to catch all exceptions, initially tried TimeoutError, it was not catching it.
            except:
                # send this message in as lyrics, when the page renders, it will print the message in place of the lyrics.
                lyrics = 'No Lyrics Found. Try checking spelling, lyrics may not be available for all songs.'
                context = {
                    # I still pass in song, because it is used in the header.
                    'song': song,
                    'lyrics': lyrics,
                }
                return render(request, "ArtistTrack_lyrics.html", context)
    # if the database already has lyrics for the given song, print the lyrics from the database.
    else:
        lyrics = song.lyrics
        context = {
            'song': song,
            'lyrics': lyrics,
        }
        return render(request, "ArtistTrack_lyrics.html", context)
        
 ```
 
### Front End Development
For the library page, I, utilized flexbox in CSS to display containers of playlists that could potentially contain hundreds of songs all together on the same page. I utilized bootstrap as well as my own CSS to get the look I wanted. Here is the navbar template. I utilized Django Template Language again, to keep the current page highlighted on the navbar.

![image](https://user-images.githubusercontent.com/80490144/121760980-4c82eb00-cae2-11eb-9abc-2732edce6b88.png)

```

<nav class="navbar navbar-expand-lg navbar-dark nav-opaque-container">
  <a class="navbar-brand" href="{% url 'at_home' %}">ArtistTrack</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNavAltMarkup" aria-controls="navbarNavAltMarkup" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>
  <div class="collapse navbar-collapse" id="navbarNavAltMarkup">
    <div class="navbar-nav">
<!--   this gets the url name and saves it as url_name-->
      {% with url_name=request.resolver_match.url_name %}
<!--   if url_name = current url, style nav button as active (bootstrap class)-->
      <a class="nav-item nav-link {% if url_name == 'at_library' %}active{% endif %}" href="{% url 'at_library' %}">Library</a>
      <a class="nav-item nav-link {% if url_name == 'at_all_songs' %}active{% endif %}" href="{% url 'at_all_songs' %}">Songs</a>
      <a class="nav-item nav-link {% if url_name == 'add_song' %}active{% endif %}" href="{% url 'add_song' %}">Add Song</a>
      <a class="nav-item nav-link {% if url_name == 'add_playlist' %}active{% endif %}" href="{% url 'add_playlist' %}">Add Playlist</a>
      {% endwith %}
    </div>
  </div>
</nav>

```

### Skills
* Working with Agile/Scrum methodology, including breaking down a project into easily digestable parts, and working on the parts. 
* Explaining code with daily standups and daily reports,  
* Communicating with developers and project managers to ask for help, guidance, and to manage time. 
* Using Git to manage edits, checkout branches, make pull requests, and rolling back when necessary. 
* Using view functions to get data from the databse, and send it to a view for display. 
* Utilizing django messages to display a confirm message. 
Put this in the view function. 
```
    messages.add_message(request, messages.SUCCESS,  'Song "{}" Saved To Library'.format(song_name))
```
Use this in the template
```
    {% if messages %}
    <ul class="messages text-white">
        {% for message in messages %}
        <li {% if message.tags %} class="{{ message.tags }}" {% endif %}>{{ message }}</li>
        {% endfor %}
    </ul>
    {% endif %}
```
