
# Django Signals and Custom Python Classes

This README covers key concepts about Django signals and includes a custom Rectangle class implementation.

## Django Signals

### 1. Synchronous Execution of Django Signals

By default, Django signals are executed synchronously. Here's a code snippet that demonstrates this:

```python
from django.db import models
from django.db.models.signals import post_save
from django.dispatch import receiver
import time

class TestModel(models.Model):
    name = models.CharField(max_length=100)

@receiver(post_save, sender=TestModel)
def slow_signal_handler(sender, instance, created, **kwargs):
    print(f"Signal handler started for {instance.name}")
    time.sleep(2)  # Simulate a time-consuming operation
    print(f"Signal handler finished for {instance.name}")

def test_signal_execution():
    start_time = time.time()
    
    print("Creating first object...")
    TestModel.objects.create(name="Object 1")
    print(f"First object created at {time.time() - start_time:.2f} seconds")
    
    print("Creating second object...")
    TestModel.objects.create(name="Object 2")
    print(f"Second object created at {time.time() - start_time:.2f} seconds")
    
    print(f"Total execution time: {time.time() - start_time:.2f} seconds")

# Run the test function
test_signal_execution()
```

This code proves that signals are executed synchronously because the creation of the second object is delayed until after the signal handler for the first object completes.

### 2. Thread Behavior of Django Signals

Django signals run in the same thread as the caller. Here's a code snippet that demonstrates this:

```python
from django.db import models
from django.db.models.signals import post_save
from django.dispatch import receiver
import threading
import time

class TestModel(models.Model):
    name = models.CharField(max_length=100)

@receiver(post_save, sender=TestModel)
def signal_handler(sender, instance, created, **kwargs):
    current_thread = threading.current_thread()
    print(f"Signal handler started for {instance.name} in thread: {current_thread.name}")
    time.sleep(1)  # Simulate some work
    print(f"Signal handler finished for {instance.name} in thread: {current_thread.name}")

def create_object(name):
    current_thread = threading.current_thread()
    print(f"Creating object {name} in thread: {current_thread.name}")
    TestModel.objects.create(name=name)
    print(f"Object {name} created in thread: {current_thread.name}")

def test_signal_thread_behavior():
    # Create objects in the main thread
    create_object("MainThread1")
    create_object("MainThread2")

    # Create objects in separate threads
    thread1 = threading.Thread(target=create_object, args=("Thread1",))
    thread2 = threading.Thread(target=create_object, args=("Thread2",))

    thread1.start()
    thread2.start()

    thread1.join()
    thread2.join()

if __name__ == "__main__":
    test_signal_thread_behavior()
```

This code proves that signals run in the same thread as the caller because the thread names in the signal handler match those of the object creation.

### 3. Transaction Behavior of Django Signals

By default, Django signals run in the same database transaction as the caller. Here's a code snippet that demonstrates this:

```python
from django.db import models, transaction
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.core.exceptions import ObjectDoesNotExist

class MainModel(models.Model):
    name = models.CharField(max_length=100)

class RelatedModel(models.Model):
    main = models.ForeignKey(MainModel, on_delete=models.CASCADE)
    value = models.IntegerField()

@receiver(post_save, sender=MainModel)
def create_related_object(sender, instance, created, **kwargs):
    if created:
        print(f"Signal handler: Creating RelatedModel for {instance.name}")
        RelatedModel.objects.create(main=instance, value=42)
        print(f"Signal handler: RelatedModel created for {instance.name}")
        if instance.name == "RollbackTest":
            raise Exception("Simulated error in signal handler")

def test_transaction_behavior():
    print("Test 1: Successful transaction")
    with transaction.atomic():
        main_obj = MainModel.objects.create(name="SuccessTest")
    print(f"MainModel created: {MainModel.objects.filter(name='SuccessTest').exists()}")
    print(f"RelatedModel created: {RelatedModel.objects.filter(main__name='SuccessTest').exists()}")

    print("\nTest 2: Rollback due to exception in signal handler")
    try:
        with transaction.atomic():
            MainModel.objects.create(name="RollbackTest")
    except Exception as e:
        print(f"Caught exception: {str(e)}")

    print(f"MainModel exists: {MainModel.objects.filter(name='RollbackTest').exists()}")
    print(f"RelatedModel exists: {RelatedModel.objects.filter(main__name='RollbackTest').exists()}")

    print("\nTest 3: Rollback due to exception after signal handler")
    try:
        with transaction.atomic():
            MainModel.objects.create(name="PostSignalRollbackTest")
            raise Exception("Simulated error after signal handler")
    except Exception as e:
        print(f"Caught exception: {str(e)}")

    print(f"MainModel exists: {MainModel.objects.filter(name='PostSignalRollbackTest').exists()}")
    print(f"RelatedModel exists: {RelatedModel.objects.filter(main__name='PostSignalRollbackTest').exists()}")

if __name__ == "__main__":
    test_transaction_behavior()
```

This code proves that signals run in the same transaction as the caller because when an exception is raised (either in the signal handler or after it), both the main object and the related object created in the signal handler are rolled back.

## Custom Rectangle Class

Here's an implementation of a Rectangle class that meets the specified requirements:

```python
class Rectangle:
    def __init__(self, length: int, width: int):
        self.length = length
        self.width = width

    def __iter__(self):
        yield {'length': self.length}
        yield {'width': self.width}

# Example usage:
rect = Rectangle(5, 3)

for item in rect:
    print(item)
```

This Rectangle class satisfies the given requirements:

1. It requires `length` and `width` (both integers) to be initialized.
2. It can be iterated over, yielding the length and width in the specified format.

When you run this code, you'll see output like this:

```
{'length': 5}
{'width': 3}
```

This demonstrates that the Rectangle class can be iterated over, first yielding the length and then the width in the required format.

