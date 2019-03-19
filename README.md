
# Django Forum Live Project

## Introduction
For the past two weeks I participated in a group live project developing a forum Web app in Django for The Tech Academy. This project is intended for eventual deployment and real-world commerical use. We were the first group to work on this project so we were starting nearly from scratch. I worked on several back-end stories and developed much of the base functionality of the site. Because it we were starting from scratch the Django HTML templates have minimal styling. 



## Stories
All the stories I completed over the two week period. 
![Tickets Completed][tickets]


## User Profile
This is the first ticket I worked on. I had to make a view to display the user signature and profile picture and allow them to change them. 
![Default Profile][profile1] ![Uploaded Profile Avatar][profile2]

***views.py***
```python
@login_required
@require_http_methods(['POST'])
def get_profile(request):
    # if this is a post request we need to process the form data
    if request.method == 'POST':
        # create a form instance and populate it with data from the request
        form = ProfileForm(request.POST or None, request.FILES or None)
        # check if valid
        if form.is_valid():
            # process the data in form as required
            # redirect to a new URL
            form = form.cleaned_data
            UserProfile.updateProfile(request, form)
            return HttpResponseRedirect('/')
        else:
            print(form.errors)

    else:
      sig = request.user.userprofile.Signature
      form = ProfileForm({'Signature': sig})
    context = {
      'form': form,
    }

    return render(request, 'user_profile/index.html', context)
```
***models.py***
``` python
def user_directory_path(instance, filename):
  # TODO: Upload to media/user_<id>/avatar/<filename>
  #file will be uploaded to MEDIA_ROOT/user_<id>/<filename>
  return 'avatar/user_{0}/{1}'.format(instance.User.id, filename)

class UserProfile(models.Model):
    User = models.OneToOneField(User, on_delete=models.CASCADE)
    Signature = models.CharField(max_length=200, null=True)
    Avatar = models.ImageField(upload_to = user_directory_path, blank=True, default='avatar/default-avatar.png')

    def __str__(self):
        """String for replacing the default 'UserProfile object 1' formatting """
        return '{self.User.username} Profile'

    @classmethod
    def updateProfile(self, request, form):
      # We need to see if the user is logged in before they update their profile.
      user = request.user
      if user.is_authenticated:
        userID = user.id
        sig = form['Signature']
        #if the Avatar field is empty, value will be false, otherwise ImageFile
        avatar = request.FILES['Avatar'] if 'Avatar' in request.FILES else False
        try:
          # I think we should create a UserProfile object on account registration.
          UserProfile.objects.create(User=User.objects.get(User_id=userID), Signature=sig, Avatar=avatar )
        except:
          # Otherwise update the database
          profile = UserProfile.objects.get(User_id = userID)
          profile.Signature = sig
          # if the Avatar field wasn't empty, do stuff
          if avatar != False:
            avatar_url = profile.Avatar.url
            # if the current avatar url DOES NOT match the default,
            # delete the old avatar then create the new one
            if not avatar_url.endswith('/media/avatar/default-avatar.png'):
              if os.path.isfile(settings.BASE_DIR + avatar_url):
                os.remove(settings.BASE_DIR + avatar_url)
            profile.Avatar = avatar
          profile.save()

      else:
        #  ask them to register (or login?) if their user doesn't authenticate
        HttpResponseRedirect('/register/')
```

## Friend Page
![Friend Page 1][friend1] ![Friend Page 2][friend2]
I created a page where a user could send a friend request by username, remove established friends, accept or decline incoming friend requests, and cancel outgoing friend requests. I also wanted to implement a autocomplete search function for the add friends field. I did this using jQuery and jQuery-UI.
```
# Use AJAX to display autocomplete search results in the friends page
def autocomplete(request):
  if request.is_ajax():
    user = request.user.id
    friends = FriendConnection.objects.filter(Q(ReceivingUser = user) |
                                              Q(SendingUser = user))
    sending_friends_ids = friends.values('SendingUser')
    receiving_friends_ids = friends.values('ReceivingUser')

    queryset = User.objects.exclude(Q(id__in=sending_friends_ids) | \
     Q(id__in=receiving_friends_ids)) \
    .filter(username__startswith=request.GET.get('search', None))
    list = []
    for i in queryset:
      list.append(i.username)
    data = {
      'list': list
    }
    return JsonResponse(data)



def change_friend_status(request, **kwargs):
  if request.user.is_authenticated:
    if request.method == 'POST':
      # If we aren't adding a friend, then see if we are confirming or deleting
      if not 'add_friend' in request.POST:
        user = request.user
        friendID = kwargs['id']
        friend = FriendConnection.objects.get(id=friendID)
        if 'confirm_friend' in request.POST:
          confirmed = not friend.IsConfirmed
          friend.IsConfirmed = confirmed
          friend.save(update_fields=['IsConfirmed'])
        elif 'delete_friend' in request.POST:
          friend.delete()
      # This is if we are adding a friend
      else:
        try:
          user = User.objects.get(id = kwargs['id'])
          friend = User.objects.get(username = request.POST.get('friend_name', None))
          FriendConnection.objects.create(IsConfirmed=0, SendingUser=user, ReceivingUser = friend)
        except:
          return HttpResponseBadRequest()
    return HttpResponseRedirect("/home/friends/")


class FriendListView(generic.ListView):
  model = FriendConnection
  template_name = 'forum/friend_list.html'
  context_object_name = 'friends'

  def get_queryset(self):
    userID = self.request.user

    return FriendConnection.objects.filter((Q(IsConfirmed=1) & (Q(ReceivingUser_id = userID) | Q(SendingUser_id = userID)))).all()


  def get_context_data(self, *args, **kwargs):
    context = super(FriendListView, self).get_context_data(**kwargs)
    userID = self.request.user
    context['unconfirmed_friends'] = FriendConnection.objects.filter((Q(IsConfirmed=0) &
                                          (Q(ReceivingUser_id = userID) |
                                          Q(SendingUser_id = userID))))
    return context

```

## Thread
### Create Thread
![Create Thread Page][create_thread]
I created a simple view to allow a user to create a thread and then redirect to that thread on successful thread creation.


***views.py***
```
class ThreadCreateView(CreateView):
  template_name = 'forum/thread_form.html'
  model = Thread
  form_class = ThreadCreateForm


  def form_valid(self, form):
    form = form.save(commit=False)
    form.Author = self.request.user
    form.ViewCount = 1
    form.PostCount = 0
    today = datetime.date.today().strftime('%Y-%m-%d')
    form.DateStarted = today
    form.DateUpdate = today
    form.save()
    return HttpResponseRedirect("/home/thread/{}/".format(form.id))
```
### Display Thread
I wanted to create a view that would display the thread and the form to post on the thread. This was where I learned how to use context data to pass the form and thread header to the view. 


***views.py***
```
# Display the comments of a thread
class CommentThread(generic.ListView):
    template_name = 'forum/comment_thread.html'
    context_object_name = 'comments'
    ordering = ['-DateCreated']


    def get_queryset(self, *args, **kwargs):
        pk = self.kwargs['pk']
        return Comment.objects.filter(Thread_id=pk).order_by('DateCreated').reverse


    def get_context_data(self, *args, **kwargs):
        context = super(CommentThread, self).get_context_data(**kwargs)
        pk = self.kwargs['pk']
        context['thread'] = Thread.objects.filter(id = pk).last
        context['form'] = CommentCreateForm()

        return context
```
### Upvote Thread
When I approached the upvote thread ticket I thought that we would need a way to track who upvoted which threads. I thought that we could add a field to the thread model that would hold an array of user IDs of who upvoted the thread. This is possible with PostgreSQL, but not with SQLite, which we were using. I considered validating and manipulating a charfield to create an array, but decided against it in favor of making an upvote model. The model would simply hold the IDs of the user upvoting and the thread upvoted. 

***models.py***
```
class Upvote(models.Model):
    User = models.ForeignKey(User, null=True, on_delete=models.SET_NULL)
    Thread = models.ForeignKey(Thread, on_delete=models.CASCADE)
```
<br>

***views.py***
```
def upvote_thread(request, *args, **kwargs):
  if request.user.is_authenticated:
    if request.method == 'GET':
      userID = request.user.id
      threadID = kwargs['id']
      user_upvoted_thread = Upvote.objects.filter(Thread_id=threadID, User_id=userID)
      if user_upvoted_thread.count():
        user_upvoted_thread.delete()
      else:
        Upvote.objects.create(Thread_id=threadID, User_id=userID)

      thread = Thread.objects.get(id=threadID)
      count = Upvote.objects.filter(Thread_id=threadID).count()
      thread.UpVoteCount = count
      thread.save()
      return HttpResponse(count)
  else:
    HttpResponseRedirect("{% url 'login' %}")
```
I used jQuery in the HTML template to call the view function.
```
<script type='text/javascript'>
  $("#upvotes").click(function(){
    var threadID;
    threadID = $(this).attr("data-threadID");
    $.get('{% url "Forum:upvote" thread.id %}', function(data){
      $('#upvote_count').html(data);
    });
  });
</script>
```
  
## Topics
Create a page which displays the thread topics and links to the most recently updated thread.
![Thread Topics Page][topics]
Django has the ability to craft custom SQL queries, which I used on this page. I tried to use regular Django queries, but SQLite doesn't have the ability to return distinct results by a given field. Instead, in the custom SQL query `GROUP BY Topic_id` serves that function.

***views.py***
```
# Return a dictionary of thread topics
class TopicsView(generic.ListView):
    model = Topic
    template_name = 'topics.html'

    def get_queryset(self):
        return dictfetchall('''SELECT Forum_thread.id as thread_id,
                            Forum_topic.TopicTitle as "topic_title",
                            ThreadTitle as thread_title,
                            COUNT(Forum_thread.id) as thread_count,
                            MAX(Forum_thread.DateUpdate) as update_date
                            FROM Forum_thread
                            INNER JOIN Forum_topic
                            ON Forum_topic.id = Forum_thread.Topic_id
                            GROUP BY Topic_id
                            ORDER BY Forum_topic.id ASC''')

# Helper Fucntions

# This makes a dictionary out of a custom SQL query
def dictfetchall(query):
    "Return all rows from a cursor as a dict"
    with connection.cursor() as cursor:
        cursor.execute(query)
        columns = [col[0] for col in cursor.description]
        return [
            dict(zip(columns, row))
            for row in cursor.fetchall()
        ]
```

[topics]:https://i.ibb.co/m90pkhL/topics.png
[tickets]: https://i.ibb.co/6bZ0Wdn/tickets.png
[thread]:https://i.ibb.co/Y0nk99T/thread.png
[profile2]:https://i.ibb.co/6WX8y8R/profile2.png
[profile1]:https://i.ibb.co/p1T4yxp/profile1.png
[create_thread]:https://i.ibb.co/c2rDrkz/Create-thread.png
[friend1]:https://i.ibb.co/xCH8Jzv/add-friend.png
[friend2]: https://i.ibb.co/GWgPtwM/add-friend-dropdown.png
