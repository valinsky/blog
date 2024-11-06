---
date: '2024-11-05'
title: 'Beware of Python''s Mutable Default Arguments Anti-Pattern'
draft: false
comments: true
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
In Python, using mutable default arguments in function definitions is considered an anti-pattern because it can lead to unintended behaviour.

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

The recommended approach is to default `my_list` to `None` and conditionally assign the empty list object inside the function body. This way, Python creates a new list object on every function call.
```python
def append_to_list(number, my_list=None):
    if my_list is None:
        my_list = []
    my_list.append(number)
    return my_list

list1 = append_to_list(1)
print(list1)

list2 = append_to_list(2)
print(list2)
```

Now `list1` and `list2` are separate list objects and the output is:
```
[1]
[2]
```


### My own experience

I had to deal with a similar situation when writing an [AWS Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) that would read an S3 file in chunks, validate the data, and then perform an S3 multipart upload. The code looked something like this (simplified for illustration purposes):
```python
class S3MultipartUploader:
    def __init__(self, parts=[]):
        self.parts = parts
        self.s3_client = boto3.client('s3')

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

The code defines a `S3MultipartUploader` class that uploads data in chunks, and then completes the upload. And it was working great, until it didn't. At random times the Lambda would throw an error with the following message:
`"An error occurred (InvalidPart) when calling the CompleteMultipartUpload operation: One or more of the specified parts could not be found."`. After some debugging, I discovered that the issue originated from a combination of mutable default arguments and [Lambda container reusage](https://aws.amazon.com/blogs/compute/container-reuse-in-lambda/).

The `__init__` method was using a mutable default argument by setting `parts` to an empty list. When the Lambda had a cold start in a new container, `multipart_uploader` was instantiated with `parts` as an empty list and there were no issues. However, when the Lambda was reusing a previous container, `parts` retained data from that container. As a result, `multipart_uploader` was instantiated with leftover data from a previous execution context, similar to our first example where `list2 = append_to_list(2)` was reusing a non-empty `my_list` that was mutated in a previous function call. This led to `multipart_uploader` attempting to complete the upload with parts that didn't exist in the current execution context, causing the Lambda to fail.

The fix was easy:
```python
def __init__(self, parts=None):
    if parts is None:
        parts = []
```

### Conclusion

Mutable default arguments are rarely needed, if ever. If you want your code to behave as you intend, avoid hours of debugging unexpected behavior, and ensure your peace of mind, it's best to avoid them and follow best practices. The alternative presented above is a reliable, battle-tested approach.
