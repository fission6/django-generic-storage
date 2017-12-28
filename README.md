# django-generic-storage
Generic Storage Class for Django

Control Django Storage per Model Field based on Settings Configuration

```
from django.conf import settings
from django.utils.module_loading import import_string
from django.utils.deconstruct import deconstructible

def get_storage_class(import_path=None):
    return import_string(import_path or settings.DEFAULT_FILE_STORAGE)

@deconstructible
class GenericStorage(object):
    """
    Generic storage class which takes in a key and looks up a storage defined
    in STORAGES in django settings. If setting does not exist then use the default
    storage.
    """
    def __init__(self, storage_key):
        self.storage_key = storage_key
        self._storage = self.build_storage()

    def __getattr__(self, name):
        """Proxy to underlying storage class"""
        return getattr(self._storage, name)

    def get_storage_from_settings(self):
        """Fetch backend from settings"""
        storage_key = self.storage_key
        storage_backend = None

        if storage_key and hasattr(settings, 'STORAGES'):
            try:
                storage_entry = settings.STORAGES[self.storage_key]
            except KeyError:
                msg = 'Could not find storage {}'.format(self.storage_key)
                raise KeyError(msg)
            storage_backend = storage_entry.get('BACKEND')

        return storage_backend

    def build_storage(self):
        """Construct Storage"""
        storage_backend = self.get_storage_from_settings()
        return get_storage_class(storage_backend)()

```

```
settings

STORAGES = {
    'public-media': {
        'BACKEND': 'whatever.storages.backends.s3boto3.PublicMediaStorage',
    },
    'private-media': {
        'BACKEND': 'whatever.storages.backends.s3boto3.PrivateMediaStorage',
    }
}

```
```
storages
from storages.backends.s3boto3 import S3Boto3Storage


class PublicMediaStorage(S3Boto3Storage):
    """S3 Public Storage"""
    bucket_name = 'media.whatever.com'
    default_acl = 'private' # made public through aws bucket policy
    file_overwrite = False
    location = 'media'
    querystring_auth = False
    region_name = 'us-west-2'


class PrivateMediaStorage(S3Boto3Storage):
    """S3 Private Storage"""
    bucket_name = 'media.whatever.com'
    default_acl = 'private'
    file_overwrite = False
    location = 'media'
    querystring_expire = 60 * 60
    region_name = 'us-west-2'
```

```
example

document = models.FileField(upload_to,...,storage=GenericStorage('public-media'))
```
