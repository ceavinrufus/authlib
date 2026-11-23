Added
~~~~~

- The OpenID Connect registration can verify the ``sector_identifier_uri``
  document, per OpenID Connect Registration §5. Implement
  ``ClientMetadataClaims.fetch_sector_identifier_document`` in a subclass to
  return the document. The registration and the client configuration endpoints
  reject a client whose ``redirect_uris`` are not all in the document with
  ``invalid_client_metadata``. :issue:`1073`
