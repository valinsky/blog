---
date: '2024-11-01'
title: 'Beware of Python''s mutable default arguments anti-pattern'
draft: true
comments: false
showToc: false
TocOpen: false
hidemeta: false
# description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: false
ShowWordCount: false
ShowRssButtonInSectionTermList: false
UseHugoToc: false
---
In Python, it's not advised to use mutable default arguments in function definitions because the code can have unintended behaviour.

To showcase this let's write a simple function that appends a number to a list.
```python
def append_to_list(number, my_list=[]):
    my_list.append(number)
    return my_list

list1 = append_to_list(1)
print(list1)

list2 = append_to_list(2)
print(list2)
```

One might expect that a new empty list is created on each function call, with `list1` and `list2` being separate lists, and the output to be:
```
[1]
[2]
```

What actually happens is that Python creates a new list only once, when the function is defined. It then reuses the same list on subsequent calls, so `list1` and `list2` are, in fact, the same list object. The actual output is:
```
[1]
[1, 2]
```

The recommended approach is to default `my_list` to `None` and assign the empty list object inside the function body. This way, Python creates a new list object on every function call. `list1` and `list2` are now separate list objects.
```python
def append_to_list(number, my_list=None):
    my_list = [] if my_list is None else my_list
    my_list.append(number)
    return my_list

list1 = append_to_list(1)
print(list1)

list2 = append_to_list(2)
print(list2)
```

The new output is:
```
[1]
[2]
```


### My own experience

I had to deal with a similar situation when writing a Lambda function that would read an S3 file in chunks and then perform an S3 multipart upload. The code looked something like this (the code has been simplified for illustration purposes):
```python
class S3MultipartUploader:
    def __init__(self, parts=[]):
        self.s3_client = boto3.client('s3')
        self.parts = parts

    def upload_part(self, part):
        part_number = len(self.parts) + 1
        response = self.s3_client.upload_part(
            Body=part, PartNumber=part_number
        )
        self.parts.append({'PartNumber': part_number, 'ETag': response['ETag'], })

    def complete_upload(self):
        return self.s3_client.complete_multipart_upload(
            MultipartUpload={'Parts': self.parts, }
        )


def lambda_handler(event, context):
    multipart_uploader = S3MultipartUploader()
    for chunk in read_s3_file():
        multipart_uploader.upload_part(chunk)
    multipart_uploader.complete_upload()
```

The code defines a `S3MultipartUploader` class that uploads data in chunks, and then completes the upload. The code worked great, until it didn't. At random times the Lambda would throw an error with the following message:
`"An error occurred (InvalidPart) when calling the CompleteMultipartUpload operation: One or more of the specified parts could not be found."`

After some debugging, I discovered that the issue originated from a combination of mutable default arguments and [Lambda container reusage](https://aws.amazon.com/blogs/compute/container-reuse-in-lambda/). The `__init__` method was defaulting `parts` to an empty list. When the Lambda had a cold start in a new container, `multipart_uploader` was instantiated with `parts` as an empty list and there were no issues. However, when the Lambda was reusing a previous container, `multipart_uploader` would be instantiated with the existing `parts` list from that container. As a result, `parts` retained data from a previous execution context, similar to our first example where `list2 = append_to_list(2)` was reusing a non-empty `my_list` from a previous call. This led to `multipart_uploader` attempting to complete the upload with parts that didn't exist for its current execution context, and the Lambda would fail.

The fix was easy:
```python
def __init__(self, parts=None):
    self.parts = [] if parts is None else parts
```