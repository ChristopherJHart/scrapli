<link rel="preload stylesheet" as="style" href="https://cdnjs.cloudflare.com/ajax/libs/10up-sanitize.css/11.0.1/sanitize.min.css" integrity="sha256-PK9q560IAAa6WVRRh76LtCaI8pjTJ2z11v0miyNNjrs=" crossorigin>
<link rel="preload stylesheet" as="style" href="https://cdnjs.cloudflare.com/ajax/libs/10up-sanitize.css/11.0.1/typography.min.css" integrity="sha256-7l/o7C8jubJiy74VsKTidCy1yBkRtiUGbVkYBylBqUg=" crossorigin>
<link rel="stylesheet preload" as="style" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/10.1.1/styles/github.min.css" crossorigin>
<script defer src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/10.1.1/highlight.min.js" integrity="sha256-Uv3H6lx7dJmRfRvH8TH6kJD1TSK1aFcwgx+mdg3epi8=" crossorigin></script>
<script>window.addEventListener('DOMContentLoaded', () => hljs.initHighlighting())</script>















#Module scrapli.driver.core.cisco_iosxr.async_driver

scrapli.driver.core.cisco_iosxr.async_driver

<details class="source">
    <summary>
        <span>Expand source code</span>
    </summary>
    <pre>
        <code class="python">
"""scrapli.driver.core.cisco_iosxr.async_driver"""
from copy import deepcopy
from io import BytesIO
from typing import Any, Callable, Dict, List, Optional, Union

from scrapli.driver import AsyncNetworkDriver
from scrapli.driver.core.cisco_iosxr.base_driver import FAILED_WHEN_CONTAINS, PRIVS
from scrapli.driver.network.base_driver import PrivilegeLevel


async def iosxr_on_open(conn: AsyncNetworkDriver) -> None:
    """
    IOSXRDriver default on_open callable

    Args:
        conn: NetworkDriver object

    Returns:
        None

    Raises:
        N/A

    """
    await conn.acquire_priv(desired_priv=conn.default_desired_privilege_level)
    await conn.send_command(command="terminal length 0")
    await conn.send_command(command="terminal width 512")


async def iosxr_on_close(conn: AsyncNetworkDriver) -> None:
    """
    IOSXRDriver default on_close callable

    Args:
        conn: NetworkDriver object

    Returns:
        None

    Raises:
        N/A
    """
    await conn.acquire_priv(desired_priv=conn.default_desired_privilege_level)
    conn.channel.write(channel_input="exit")
    conn.channel.send_return()


class AsyncIOSXRDriver(AsyncNetworkDriver):
    def __init__(
        self,
        host: str,
        privilege_levels: Optional[Dict[str, PrivilegeLevel]] = None,
        default_desired_privilege_level: str = "privilege_exec",
        port: int = 22,
        auth_username: str = "",
        auth_password: str = "",
        auth_private_key: str = "",
        auth_private_key_passphrase: str = "",
        auth_strict_key: bool = True,
        auth_bypass: bool = False,
        timeout_socket: float = 15.0,
        timeout_transport: float = 30.0,
        timeout_ops: float = 30.0,
        comms_return_char: str = "\n",
        comms_ansi: Optional[bool] = None,
        ssh_config_file: Union[str, bool] = False,
        ssh_known_hosts_file: Union[str, bool] = False,
        on_init: Optional[Callable[..., Any]] = None,
        on_open: Optional[Callable[..., Any]] = None,
        on_close: Optional[Callable[..., Any]] = None,
        transport: str = "system",
        transport_options: Optional[Dict[str, Any]] = None,
        channel_log: Union[str, bool, BytesIO] = False,
        channel_lock: bool = False,
        logging_uid: str = "",
        auth_secondary: str = "",
        failed_when_contains: Optional[List[str]] = None,
        textfsm_platform: str = "cisco_xr",
        genie_platform: str = "iosxr",
    ):
        """
        IOSXRDriver Object

        Please see `scrapli.driver.base.base_driver.Driver` for all "base driver" arguments!

        # noqa: DAR101

        Args:
            privilege_levels: optional user provided privilege levels, if left None will default to
                scrapli standard privilege levels
            default_desired_privilege_level: string of name of default desired priv, this is the
                priv level that is generally used to disable paging/set terminal width and things
                like that upon first login, and is also the priv level scrapli will try to acquire
                for normal "command" operations (`send_command`, `send_commands`)
            auth_secondary: password to use for secondary authentication (enable)
            on_open: callable that accepts the class instance as its only argument. this callable,
                if provided, is executed immediately after authentication is completed. Common use
                cases for this callable would be to disable paging or accept any kind of banner
                message that prompts a user upon connection
            on_close: callable that accepts the class instance as its only argument. this callable,
                if provided, is executed immediately prior to closing the underlying transport.
                Common use cases for this callable would be to save configurations prior to exiting,
                or to logout properly to free up vtys or similar.
            textfsm_platform: string name of textfsm parser platform
            genie_platform: string name of cisco genie parser platform
            failed_when_contains: List of strings that indicate a command/config has failed

        Returns:
            N/A  # noqa: DAR202

        Raises:
            N/A

        """
        if privilege_levels is None:
            privilege_levels = deepcopy(PRIVS)

        if on_open is None:
            on_open = iosxr_on_open
        if on_close is None:
            on_close = iosxr_on_close

        if failed_when_contains is None:
            failed_when_contains = FAILED_WHEN_CONTAINS.copy()

        super().__init__(
            host=host,
            port=port,
            auth_username=auth_username,
            auth_password=auth_password,
            auth_private_key=auth_private_key,
            auth_private_key_passphrase=auth_private_key_passphrase,
            auth_strict_key=auth_strict_key,
            auth_bypass=auth_bypass,
            timeout_socket=timeout_socket,
            timeout_transport=timeout_transport,
            timeout_ops=timeout_ops,
            comms_return_char=comms_return_char,
            comms_ansi=comms_ansi,
            ssh_config_file=ssh_config_file,
            ssh_known_hosts_file=ssh_known_hosts_file,
            on_init=on_init,
            on_open=on_open,
            on_close=on_close,
            transport=transport,
            transport_options=transport_options,
            channel_log=channel_log,
            channel_lock=channel_lock,
            logging_uid=logging_uid,
            privilege_levels=privilege_levels,
            default_desired_privilege_level=default_desired_privilege_level,
            auth_secondary=auth_secondary,
            failed_when_contains=failed_when_contains,
            textfsm_platform=textfsm_platform,
            genie_platform=genie_platform,
        )

    async def _abort_config(self) -> None:
        """
        Abort IOSXR configuration session

        Args:
            N/A

        Returns:
            None

        Raises:
            N/A

        """
        await self.channel.send_input(channel_input="abort")
        self._current_priv_level = self.privilege_levels["privilege_exec"]
        </code>
    </pre>
</details>



## Functions

    

#### iosxr_on_close
`iosxr_on_close(conn: scrapli.driver.network.async_driver.AsyncNetworkDriver) ‑> NoneType`

```text
IOSXRDriver default on_close callable

Args:
    conn: NetworkDriver object

Returns:
    None

Raises:
    N/A
```




    

#### iosxr_on_open
`iosxr_on_open(conn: scrapli.driver.network.async_driver.AsyncNetworkDriver) ‑> NoneType`

```text
IOSXRDriver default on_open callable

Args:
    conn: NetworkDriver object

Returns:
    None

Raises:
    N/A
```




## Classes

### AsyncIOSXRDriver


```text
IOSXRDriver Object

Please see `scrapli.driver.base.base_driver.Driver` for all "base driver" arguments!

# noqa: DAR101

Args:
    privilege_levels: optional user provided privilege levels, if left None will default to
        scrapli standard privilege levels
    default_desired_privilege_level: string of name of default desired priv, this is the
        priv level that is generally used to disable paging/set terminal width and things
        like that upon first login, and is also the priv level scrapli will try to acquire
        for normal "command" operations (`send_command`, `send_commands`)
    auth_secondary: password to use for secondary authentication (enable)
    on_open: callable that accepts the class instance as its only argument. this callable,
        if provided, is executed immediately after authentication is completed. Common use
        cases for this callable would be to disable paging or accept any kind of banner
        message that prompts a user upon connection
    on_close: callable that accepts the class instance as its only argument. this callable,
        if provided, is executed immediately prior to closing the underlying transport.
        Common use cases for this callable would be to save configurations prior to exiting,
        or to logout properly to free up vtys or similar.
    textfsm_platform: string name of textfsm parser platform
    genie_platform: string name of cisco genie parser platform
    failed_when_contains: List of strings that indicate a command/config has failed

Returns:
    N/A  # noqa: DAR202

Raises:
    N/A
```

<details class="source">
    <summary>
        <span>Expand source code</span>
    </summary>
    <pre>
        <code class="python">
class AsyncIOSXRDriver(AsyncNetworkDriver):
    def __init__(
        self,
        host: str,
        privilege_levels: Optional[Dict[str, PrivilegeLevel]] = None,
        default_desired_privilege_level: str = "privilege_exec",
        port: int = 22,
        auth_username: str = "",
        auth_password: str = "",
        auth_private_key: str = "",
        auth_private_key_passphrase: str = "",
        auth_strict_key: bool = True,
        auth_bypass: bool = False,
        timeout_socket: float = 15.0,
        timeout_transport: float = 30.0,
        timeout_ops: float = 30.0,
        comms_return_char: str = "\n",
        comms_ansi: Optional[bool] = None,
        ssh_config_file: Union[str, bool] = False,
        ssh_known_hosts_file: Union[str, bool] = False,
        on_init: Optional[Callable[..., Any]] = None,
        on_open: Optional[Callable[..., Any]] = None,
        on_close: Optional[Callable[..., Any]] = None,
        transport: str = "system",
        transport_options: Optional[Dict[str, Any]] = None,
        channel_log: Union[str, bool, BytesIO] = False,
        channel_lock: bool = False,
        logging_uid: str = "",
        auth_secondary: str = "",
        failed_when_contains: Optional[List[str]] = None,
        textfsm_platform: str = "cisco_xr",
        genie_platform: str = "iosxr",
    ):
        """
        IOSXRDriver Object

        Please see `scrapli.driver.base.base_driver.Driver` for all "base driver" arguments!

        # noqa: DAR101

        Args:
            privilege_levels: optional user provided privilege levels, if left None will default to
                scrapli standard privilege levels
            default_desired_privilege_level: string of name of default desired priv, this is the
                priv level that is generally used to disable paging/set terminal width and things
                like that upon first login, and is also the priv level scrapli will try to acquire
                for normal "command" operations (`send_command`, `send_commands`)
            auth_secondary: password to use for secondary authentication (enable)
            on_open: callable that accepts the class instance as its only argument. this callable,
                if provided, is executed immediately after authentication is completed. Common use
                cases for this callable would be to disable paging or accept any kind of banner
                message that prompts a user upon connection
            on_close: callable that accepts the class instance as its only argument. this callable,
                if provided, is executed immediately prior to closing the underlying transport.
                Common use cases for this callable would be to save configurations prior to exiting,
                or to logout properly to free up vtys or similar.
            textfsm_platform: string name of textfsm parser platform
            genie_platform: string name of cisco genie parser platform
            failed_when_contains: List of strings that indicate a command/config has failed

        Returns:
            N/A  # noqa: DAR202

        Raises:
            N/A

        """
        if privilege_levels is None:
            privilege_levels = deepcopy(PRIVS)

        if on_open is None:
            on_open = iosxr_on_open
        if on_close is None:
            on_close = iosxr_on_close

        if failed_when_contains is None:
            failed_when_contains = FAILED_WHEN_CONTAINS.copy()

        super().__init__(
            host=host,
            port=port,
            auth_username=auth_username,
            auth_password=auth_password,
            auth_private_key=auth_private_key,
            auth_private_key_passphrase=auth_private_key_passphrase,
            auth_strict_key=auth_strict_key,
            auth_bypass=auth_bypass,
            timeout_socket=timeout_socket,
            timeout_transport=timeout_transport,
            timeout_ops=timeout_ops,
            comms_return_char=comms_return_char,
            comms_ansi=comms_ansi,
            ssh_config_file=ssh_config_file,
            ssh_known_hosts_file=ssh_known_hosts_file,
            on_init=on_init,
            on_open=on_open,
            on_close=on_close,
            transport=transport,
            transport_options=transport_options,
            channel_log=channel_log,
            channel_lock=channel_lock,
            logging_uid=logging_uid,
            privilege_levels=privilege_levels,
            default_desired_privilege_level=default_desired_privilege_level,
            auth_secondary=auth_secondary,
            failed_when_contains=failed_when_contains,
            textfsm_platform=textfsm_platform,
            genie_platform=genie_platform,
        )

    async def _abort_config(self) -> None:
        """
        Abort IOSXR configuration session

        Args:
            N/A

        Returns:
            None

        Raises:
            N/A

        """
        await self.channel.send_input(channel_input="abort")
        self._current_priv_level = self.privilege_levels["privilege_exec"]
        </code>
    </pre>
</details>


#### Ancestors (in MRO)
- scrapli.driver.network.async_driver.AsyncNetworkDriver
- scrapli.driver.generic.async_driver.AsyncGenericDriver
- scrapli.driver.base.async_driver.AsyncDriver
- scrapli.driver.base.base_driver.BaseDriver
- scrapli.driver.generic.base_driver.BaseGenericDriver
- scrapli.driver.network.base_driver.BaseNetworkDriver
#### Class variables

    
`auth_secondary: str`




    
`failed_when_contains: List[str]`




    
`genie_platform: str`




    
`logger: logging.LoggerAdapter`




    
`privilege_levels: Dict[str, scrapli.driver.network.base_driver.PrivilegeLevel]`




    
`textfsm_platform: str`