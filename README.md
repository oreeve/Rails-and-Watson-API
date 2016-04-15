# Rails-and-Watson-API

The Watson Document Conversion API allows you to input a Word, PDF, or HTML document and convert it to plain text, HTML, or JSON format. In my app, I wanted to convert an uploaded PDF into JSON. Because the Watson API has no Ruby documentation, and Ruby doesn't natively support uploading a file through an HTTP request, I used the [faraday](https://github.com/lostisland/faraday) gem to format the HTTP request. I also used [carrierwave](https://github.com/carrierwaveuploader/carrierwave) for file uploading.

In my database I have a model called `Assignment` which has numerous properties, one of which is the property `file` which is a string. 

When creating a new assignment on the `/assignments/new` page, the file is saved via the following form field:

```ruby
<div class="file">
  <%= f.label :file, "File (PDF Only)" %>
  <%= f.file_field :file, class: "filebutton"%>
</div>
```

My converted document would be displayed on the Assignment "show" page (`/assignments/:id`). This is where the call to the Watson API would take place. In the Assignments controller, I had the following code:

```ruby
def show
  @assignment = Assignment.find(params[:id])
  @document = WatsonApi.new(@assignment.file)
end
```

I created a model wrapper, `WatsonApi`, to call the API in the file `app/models/watson_api.rb`.

