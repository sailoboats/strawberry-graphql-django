# Strawberry GraphQL Django extension

[![CI](https://github.com/la4de/strawberry-graphql-django/actions/workflows/main.yml/badge.svg)](https://github.com/la4de/strawberry-graphql-django/actions/workflows/main.yml)
[![PyPI](https://img.shields.io/pypi/v/strawberry-graphql-django)](https://pypi.org/project/strawberry-graphql-django/)
[![PyPI - Downloads](https://img.shields.io/pypi/dm/strawberry-graphql-django)](https://pypi.org/project/strawberry-graphql-django/)

This library provides helpers to generate fields, mutations and resolvers for Django models.

Installing strawberry-graphql-django packet from the python package repository.
```shell
pip install strawberry-graphql-django
```

## Example project files

See example Django project [examples/django](examples/django).

models.py:
```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=50)
    age = models.IntegerField()
    groups = models.ManyToManyField('Group', related_name='users')

class Group(models.Model):
    name = models.CharField(max_length=50)
```

schema.py:
```python
from typing import List
import strawberry
from strawberry_django import ModelResolver, ModelPermissions
from .models import User, Group

class GroupResolver(ModelResolver):
    model = Group
    fields = ['name', 'users']

    # only users who have group permissions can access and modify groups
    permissions_classes = [ModelPermissions]

    # queryset filtering
    def get_queryset(self):
        qs = super().get_queryset()
        # only super users can access groups
        if not self.request.user.is_superuser:
            qs = qs.none()
        return qs

class UserResolver(ModelResolver):
    model = User
    @strawberry.field
    def age_in_months(self, info, root) -> int:
        return root.age * 12

    # "ModelResolver.output_type" is a "strawberry.type".
    # So we can use it in any "strawberry.field".
    @strawberry.field
    def groups(self, info, root) -> List[GroupResolver.output_type]:
        if not info.context["request"].user.is_superuser:
            return root.groups.none()
        return root.groups.all()

@strawberry.type
class Query(UserResolver.query(), GroupResolver.query()):
    pass

@strawberry.type
class Mutation(UserResolver.mutation(), GroupResolver.mutation()):
    pass

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

urls.py:
```python
from django.urls import include, path
from strawberry.django.views import GraphQLView
from .schema import schema

urlpatterns = [
    path('graphql', GraphQLView.as_view(schema=schema)),
]
```

Add models and schema. Create database. Start development server.
```shell
manage.py makemigrations
manage.py migrate
manage.py runserver
```

## Mutations and Queries

Open http://localhost:8000/graphql and start testing.

Create new user.
```
mutation {
  createUser(data: {name: "my user", age: 20}) {
    id
  }
}
```

Make first queries.
```
query {
  user(id: 1) {
    name
    age
    groups {
        name
    }
  }
  users(filters: ["name__contains=my", "!age__gt=60"]) {
    id
    name
    ageInMonths
  }
}
```

Update user data.
```
mutation {
  updateUsers(data: {name: "new name"}, filters: ["id=1"]) {
    id
    name
  }
}
```

Finally delete user.
```
mutation {
  deleteUsers(filters: ["id=1"]) {
    id
  }
}
```

## Django authentication examples


schema.py:
```
class IsAuthenticated(strawberry.BasePermission):
    def has_permission(self, source: Any, info: Info, **kwargs) -> bool:
        self.message = "Not authenticated"
        return info.context.request.user.is_authenticated


class UserResolver(ModelResolver):
    model = User
    fields = (
        "id",
        "first_name",
        "last_name",
    )


@strawberry.type
class Query:
    @strawberry.field(permission_classes=[IsAuthenticated])
    def current_user(self, info: Info) -> UserResolver.output_type:
        return info.context.request.user


@strawberry.type
class Mutation:
    @strawberry.mutation(description="Login user to the current session.")
    def login(self, info: Info, username: str, password: str) -> UserResolver.output_type:
        request = info.context.request
        user = auth.authenticate(request, username=username, password=password)
        auth.login(request, user)
        return user

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

Login with:
```
mutation {
  login(username:"myuser", password:"mypassword") {
    id
    firstName
    lastName
  }
}
```

Get current user with:
```
query {
  currentUser {
    id
    firstName
    lastName
  }
}
```

## Running unit tests
```
poetry install
poetry run pytest
```

## Contributing

I would be more than happy to get pull requests, improvement ideas or any feedback from you.
