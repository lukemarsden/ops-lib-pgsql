Juju Operator Framework Charm Interface for PostgreSQL Relations
================================================================

/!\ Alpha. Everything here subject to change based on feedback and
emerging consensus.

To use this interface in your Juju Operator Framework charm, first
install it into your git branch:

```
git submodule add git+ssh://git.launchpad.net/~stub/interface-pgsql/+git/operator mod/interface-pgsql
mkdir lib/interface
ln -s ../../mod/interface-pgsql/pgsql lib/interface/
```

Your charm needs to declare its use of the interface in its `metadata.yaml` file:

```yaml
requires:
  db:
    interface: pgsql
    limit: 1  # Most charms only handle a single PostgreSQL Application.
```


Your charm needs to bootstrap it and handle events:

```python
from interface import pgsql


class MyCharm(ops.charm.CharmBase):
    _state = ops.framework.StoredState()

    def __init__(self, *args):
        super().__init__(*args)
        self._state.set_default(db_conn_str=None, db_uri=None, db_ro_uris=[])
        self.db = pgsql.PostgreSQLClient(self, 'db')  # 'db' relation in metadata.yaml
        self.framework.observe(self.db.on.database_relation_joined, self._on_database_relation_joined)
        self.framework.observe(self.db.on.master_changed, self._on_master_changed)
        self.framework.observe(self.db.on.standby_changed, self._on_standby_changed)

    def _on_database_relation_joined(self, event: pgsql.DatabaseRelationJoinedEvent):
        if self.model.unit.is_leader():
            # Provide requirements to the PostgreSQL server.
            event.database = 'mydbname'  # Request database named mydbname
            event.extensions = ['citext']  # Request the citext extension installed
        elif event.database != 'mydbname':
            # Leader has not yet set requirements. Defer, incase this unit
            # becomes leader and needs to perform that operation.
            event.defer()
            return

    def _on_master_changed(self, event: pgsql.MasterChangedEvent):
        if event.database != 'mydbname':
            # Leader has not yet set requirements. Wait until next event,
            # or risk connecting to an incorrect database.
            return
        
        # The connection to the primary database has been created,
        # changed or removed. More specific events are available, but
        # most charms will find it easier to just handle the Changed
        # events. event.master is None if the master database is not
        # available, or a pgsql.ConnectionString instance.
        self._state.db_conn_str = None if event.master is None else event.master.conn_str
        self._state.db_uri = None if event.master is None else event.master.uri

        # You probably want to emit an event here or call a setup routine to
        # do something useful with the libpq connection string or URI now they
        # are available.

    def _on_standby_changed(self, event: pgsql.StandbyChangedEvent):
        if event.database != 'mydbname':
            # Leader has not yet set requirements. Wait until next event,
            # or risk connecting to an incorrect database.
            return

        # Charms needing access to the hot standby databases can get
        # their connection details here. Applications can scale out
        # horizontally if they can make use of the read only hot
        # standby replica databases, rather than only use the single
        # master. event.stanbys will be an empty list if no hot standby
        # databases are available.
        self._state.db_ro_uris = [c.uri for c in event.standbys]
```
