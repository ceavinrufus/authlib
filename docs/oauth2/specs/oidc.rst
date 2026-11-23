.. _specs/oidc:

OpenID Connect 1.0
==================

.. meta::
    :description: General implementation of OpenID Connect 1.0 in Python.
        Learn how to create a OpenID Connect provider in Python.

This part of the documentation covers the specification of OpenID Connect. Learn
how to use it in :ref:`flask_oidc_server` and :ref:`django_oidc_server`.

OpenID Grants
-------------

.. module:: authlib.oidc.core.grants

.. autoclass:: OpenIDToken
    :show-inheritance:
    :members:

.. autoclass:: OpenIDCode
    :show-inheritance:
    :members:

.. autoclass:: OpenIDImplicitGrant
    :show-inheritance:
    :members:

.. autoclass:: OpenIDHybridGrant
    :show-inheritance:
    :members:

OpenID Endpoints
----------------

.. module:: authlib.oidc.core

.. autoclass:: AuthorizationSessionMixin
    :show-inheritance:
    :members: is_account_selection_needed

.. autoclass:: UserInfoEndpoint
    :show-inheritance:
    :members:

.. autoclass:: ClaimsParameter
    :show-inheritance:
    :members:

.. autofunction:: get_requested_acr_values

.. autofunction:: get_requested_sub

.. autoclass:: AuthorizationCodeMixin
    :show-inheritance:
    :members: get_claims_request

.. _oidc_request_objects:

OpenID request objects
----------------------

For JWT-secured authorization requests, use
:class:`authlib.oidc.core.OpenIDJWTAuthenticationRequest` instead of the generic
:ref:`JAR extension <specs/rfc9101>`. When the provider's
``require_request_uri_registration`` metadata is true, ``request_uri`` must
exactly match a URI returned by ``get_client_request_uris(client)``. When the
metadata is false or omitted, registration is optional.

In both cases, the application must implement ``validate_request_uri()`` with a
secure retrieval policy. Registration does not establish that a destination is
safe. The default retrieval policy rejects all URIs. Missing registration or
policy approval returns ``invalid_request_uri`` before retrieval:

.. code-block:: python

    from authlib.oidc.core import OpenIDJWTAuthenticationRequest

    class MyAuthenticationRequest(OpenIDJWTAuthenticationRequest):
        def get_server_metadata(self):
            return {
                "require_signed_request_object": True,
                "require_request_uri_registration": True,
            }

        def get_client_request_uris(self, client):
            return client.request_uris

        def validate_request_uri(self, request_uri, client):
            return is_approved_request_uri(request_uri, client)

        def resolve_client_public_key(self, client):
            return get_jwks_for_client(client)

        def get_request_object(self, request_uri):
            return fetch_approved_request_object(request_uri)

    authorization_server.register_extension(MyAuthenticationRequest())

The functions ``is_approved_request_uri``, ``get_jwks_for_client``, and
``fetch_approved_request_object`` must be implemented by the application.
Restrict network destinations, including resolved IP addresses, and enforce the
:ref:`retrieval safeguards described for JAR <specs/rfc9101>`. Registration is
checked before the custom validation policy, so an override cannot bypass a
provider's registration requirement. Inline ``request`` objects do not require
URI registration.

Without this extension, an OpenID request with a ``request`` or
``request_uri`` parameter is rejected with ``request_not_supported`` or
``request_uri_not_supported``. Per OpenID Connect Discovery §3,
``request_uri_parameter_supported`` defaults to true, so set it to false in the
provider metadata.

.. autoclass:: OpenIDJWTAuthenticationRequest
    :show-inheritance:
    :members:

OpenID Claims
-------------

.. module:: authlib.oidc.core.claims

.. autoclass:: IDToken
    :show-inheritance:
    :members:


.. autoclass:: CodeIDToken
    :show-inheritance:
    :members:


.. autoclass:: ImplicitIDToken
    :show-inheritance:
    :members:


.. autoclass:: HybridIDToken
    :show-inheritance:
    :members:

.. autoclass:: UserInfo
    :members:

Dynamic client registration
---------------------------

The `OpenID Connect Dynamic Client Registration <https://openid.net/specs/openid-connect-registration-1_0.html>`__ implementation is based on :ref:`RFC 7591: OAuth 2.0 Dynamic Client Registration Protocol <specs/rfc7591>`. To handle OIDC client registration, you can extend your RFC 7591 registration endpoint with OIDC claims:

.. code-block:: python

    from authlib.oauth2.rfc7591 import ClientMetadataClaims as OAuth2ClientMetadataClaims
    from authlib.oauth2.rfc7591 import ClientRegistrationEndpoint
    from authlib.oidc.registration import ClientMetadataClaims as OIDCClientMetadataClaims

    class MyClientRegistrationEndpoint(ClientRegistrationEndpoint):
        ...

        def get_server_metadata(self):
            ...

    authorization_server.register_endpoint(
        MyClientRegistrationEndpoint(
            claims_classes=[OAuth2ClientMetadataClaims, OIDCClientMetadataClaims]
        )
    )

Verifying the ``sector_identifier_uri``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Per `OpenID Connect Dynamic Client Registration §5
<https://openid.net/specs/openid-connect-registration-1_0.html#SectorIdentifierValidation>`__,
when a client registers a ``sector_identifier_uri``, the server fetches the
document that it references, a JSON array of ``redirect_uri`` values, and
checks that it contains every ``redirect_uris`` value of the client. Authlib
does not make network calls on the server side, so the fetch is left to your
application. Subclass :class:`~authlib.oidc.registration.ClientMetadataClaims`
and implement
:meth:`~authlib.oidc.registration.ClientMetadataClaims.fetch_sector_identifier_document`.
It receives the URI, which uses the ``https`` scheme, and returns the parsed
document, or ``None`` when it cannot be retrieved:

.. code-block:: python

    import requests

    class SectorClientMetadataClaims(OIDCClientMetadataClaims):
        def fetch_sector_identifier_document(self, uri):
            try:
                response = requests.get(uri, timeout=5, allow_redirects=False)
                response.raise_for_status()
                return response.json()
            except (requests.RequestException, ValueError):
                return None

    authorization_server.register_endpoint(
        MyClientRegistrationEndpoint(
            claims_classes=[OAuth2ClientMetadataClaims, SectorClientMetadataClaims]
        )
    )

The registration fails with ``invalid_client_metadata`` when the document is
not a JSON array of strings, or when a ``redirect_uris`` value of the client is
missing from it. The same classes can be given to a
:class:`~authlib.oauth2.rfc7592.ClientConfigurationEndpoint`, so that a client
update is verified too.

.. warning::

    The ``sector_identifier_uri`` is chosen by the client. Protect your
    implementation against server-side request forgery: use ``https`` only,
    refuse private and loopback addresses, do not follow redirects, set a
    timeout, and limit the size of the response.

When ``fetch_sector_identifier_document`` is not implemented, the document is
not verified, and only the form of the ``sector_identifier_uri`` is checked.



.. automodule:: authlib.oidc.registration
    :show-inheritance:
    :members:
