<link rel="preload stylesheet" as="style" href="https://cdnjs.cloudflare.com/ajax/libs/10up-sanitize.css/11.0.1/sanitize.min.css" integrity="sha256-PK9q560IAAa6WVRRh76LtCaI8pjTJ2z11v0miyNNjrs=" crossorigin>
<link rel="preload stylesheet" as="style" href="https://cdnjs.cloudflare.com/ajax/libs/10up-sanitize.css/11.0.1/typography.min.css" integrity="sha256-7l/o7C8jubJiy74VsKTidCy1yBkRtiUGbVkYBylBqUg=" crossorigin>
<link rel="stylesheet preload" as="style" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/10.1.1/styles/github.min.css" crossorigin>
<script defer src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/10.1.1/highlight.min.js" integrity="sha256-Uv3H6lx7dJmRfRvH8TH6kJD1TSK1aFcwgx+mdg3epi8=" crossorigin></script>
<script>window.addEventListener('DOMContentLoaded', () => hljs.initHighlighting())</script>















#Module scrapli.driver.network.async_driver

scrapli.driver.network.async_driver

<details class="source">
    <summary>
        <span>Expand source code</span>
    </summary>
    <pre>
        <code class="python">
"""scrapli.driver.network.async_driver"""
from collections import defaultdict
from io import BytesIO
from typing import Any, Callable, Dict, List, Optional, Tuple, Union

from scrapli.driver.generic import AsyncGenericDriver
from scrapli.driver.network.base_driver import BaseNetworkDriver, PrivilegeAction, PrivilegeLevel
from scrapli.exceptions import ScrapliPrivilegeError
from scrapli.response import MultiResponse, Response


class AsyncNetworkDriver(AsyncGenericDriver, BaseNetworkDriver):
    def __init__(
        self,
        host: str,
        privilege_levels: Dict[str, PrivilegeLevel],
        default_desired_privilege_level: str,
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
        textfsm_platform: str = "",
        genie_platform: str = "",
    ):
        # ensure type for comms_prompt_pattern exists before setting it in the mixin
        self.comms_prompt_pattern: str

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
        )

        self.auth_secondary = auth_secondary
        self.failed_when_contains = failed_when_contains or []
        self.textfsm_platform = textfsm_platform
        self.genie_platform = genie_platform

        self.privilege_levels = privilege_levels
        self.default_desired_privilege_level = default_desired_privilege_level
        self._priv_graph = defaultdict(set)
        self.update_privilege_levels()

    async def _escalate(self, escalate_priv: PrivilegeLevel) -> None:
        """
        Escalate to the next privilege level up

        Args:
            escalate_priv: privilege level to escalate to

        Returns:
            None

        Raises:
            N/A

        """
        self._pre_escalate(escalate_priv=escalate_priv)

        if escalate_priv.escalate_auth is True and self.auth_secondary:
            await super().send_interactive(
                interact_events=[
                    (escalate_priv.escalate, escalate_priv.escalate_prompt, False),
                    (self.auth_secondary, escalate_priv.pattern, True),
                ],
            )
        else:
            await self.channel.send_input(channel_input=escalate_priv.escalate)

    async def _deescalate(self, current_priv: PrivilegeLevel) -> None:
        """
        Deescalate to the next privilege level down

        Args:
            current_priv: current privilege level

        Returns:
            None

        Raises:
            N/A

        """
        await self.channel.send_input(channel_input=current_priv.deescalate)

    async def acquire_priv(self, desired_priv: str) -> None:
        """
        Acquire desired priv level

        Args:
            desired_priv: string name of desired privilege level see
                `scrapli.driver.<driver_category.device_type>.driver` for levels

        Returns:
            None

        Raises:
            ScrapliPrivilegeError: if desired_priv cannot be attained

        """
        self._validate_privilege_level_name(privilege_level_name=desired_priv)

        privilege_change_count = 0

        while True:
            current_prompt = await self.channel.get_prompt()
            privilege_action, target_priv = self._process_acquire_priv(
                destination_priv=desired_priv,
                current_prompt=current_prompt,
            )

            if privilege_action == PrivilegeAction.NO_ACTION:
                self._current_priv_level = target_priv
                return
            if privilege_action == PrivilegeAction.DEESCALATE:
                await self._deescalate(current_priv=target_priv)
            if privilege_action == PrivilegeAction.ESCALATE:
                await self._escalate(escalate_priv=target_priv)

            privilege_change_count += 1
            if privilege_change_count > len(self.privilege_levels) * 2:
                msg = f"Failed to acquire requested privilege level {desired_priv}"
                raise ScrapliPrivilegeError(msg)

    async def send_command(
        self,
        command: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        timeout_ops: Optional[float] = None,
    ) -> Response:
        """
        Send a command

        Super method will raise TypeError if anything but a string is passed here!

        Args:
            command: string to send to device in privilege exec mode
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed

        Returns:
            Response: Scrapli Response object

        Raises:
            N/A

        """
        if self._current_priv_level.name != self.default_desired_privilege_level:
            await self.acquire_priv(desired_priv=self.default_desired_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        response = await super().send_command(
            command=command,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            timeout_ops=timeout_ops,
        )
        self._update_response(response)

        return response

    async def send_commands(
        self,
        commands: List[str],
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send multiple commands

        Super method will raise TypeError if anything but a list of strings is passed here!

        Args:
            commands: list of strings to send to device in privilege exec mode
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER COMMAND sent, not for the total
                of the commands being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        if self._current_priv_level.name != self.default_desired_privilege_level:
            await self.acquire_priv(desired_priv=self.default_desired_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        responses = await super().send_commands(
            commands=commands,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            eager=eager,
            timeout_ops=timeout_ops,
        )

        for response in responses:
            self._update_response(response=response)

        return responses

    async def send_commands_from_file(
        self,
        file: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send command(s) from file

        Args:
            file: string path to file
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER COMMAND sent, not for the total
                of the commands being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        if self._current_priv_level.name != self.default_desired_privilege_level:
            await self.acquire_priv(desired_priv=self.default_desired_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        return await super().send_commands_from_file(
            file=file,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            eager=eager,
            timeout_ops=timeout_ops,
        )

    async def send_interactive(
        self,
        interact_events: Union[List[Tuple[str, str]], List[Tuple[str, str, bool]]],
        *,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        privilege_level: str = "",
        timeout_ops: Optional[float] = None,
    ) -> Response:
        """
        Interact with a device with changing prompts per input.

        Used to interact with devices where prompts change per input, and where inputs may be hidden
        such as in the case of a password input. This can be used to respond to challenges from
        devices such as the confirmation for the command "clear logging" on IOSXE devices for
        example. You may have as many elements in the "interact_events" list as needed, and each
        element of that list should be a tuple of two or three elements. The first element is always
        the input to send as a string, the second should be the expected response as a string, and
        the optional third a bool for whether or not the input is "hidden" (i.e. password input)

        An example where we need this sort of capability:

        '''
        3560CX#copy flash: scp:
        Source filename []? test1.txt
        Address or name of remote host []? 172.31.254.100
        Destination username [carl]?
        Writing test1.txt
        Password:

        Password:
         Sink: C0644 639 test1.txt
        !
        639 bytes copied in 12.066 secs (53 bytes/sec)
        3560CX#
        '''

        To accomplish this we can use the following:

        '''
        interact = conn.channel.send_inputs_interact(
            [
                ("copy flash: scp:", "Source filename []?", False),
                ("test1.txt", "Address or name of remote host []?", False),
                ("172.31.254.100", "Destination username [carl]?", False),
                ("carl", "Password:", False),
                ("super_secure_password", prompt, True),
            ]
        )
        '''

        If we needed to deal with more prompts we could simply continue adding tuples to the list of
        interact "events".

        Args:
            interact_events: list of tuples containing the "interactions" with the device
                each list element must have an input and an expected response, and may have an
                optional bool for the third and final element -- the optional bool specifies if the
                input that is sent to the device is "hidden" (ex: password), if the hidden param is
                not provided it is assumed the input is "normal" (not hidden)
            failed_when_contains: list of strings that, if present in final output, represent a
                failed command/interaction
            privilege_level: name of the privilege level to operate in
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER COMMAND sent, not for the total
                of the commands being sent!

        Returns:
            Response: scrapli Response object

        Raises:
            N/A

        """
        if privilege_level:
            self._validate_privilege_level_name(privilege_level_name=privilege_level)
            resolved_privilege_level = privilege_level
        else:
            resolved_privilege_level = self.default_desired_privilege_level

        if self._current_priv_level.name != resolved_privilege_level:
            await self.acquire_priv(desired_priv=resolved_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        # type hint is due to the TimeoutModifier wrapper returning `Any` so that we dont anger the
        # asyncio parts (which will get an awaitable not a Response returned)
        response: Response = await super().send_interactive(
            interact_events=interact_events,
            failed_when_contains=failed_when_contains,
            timeout_ops=timeout_ops,
        )
        self._update_response(response=response)

        return response

    async def _abort_config(self) -> None:
        """
        Abort a configuration operation/session if applicable (for config sessions like junos/iosxr)

        Args:
            N/A

        Returns:
            None

        Raises:
            N/A

        """

    async def send_configs(
        self,
        configs: List[str],
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        privilege_level: str = "",
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send configuration(s)

        Args:
            configs: list of strings to send to device in config mode
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution; aborts configuration session if applicable (iosxr/junos or
                eos/nxos if using a configuration session)
            privilege_level: name of configuration privilege level/type to acquire; this is platform
                dependent, so check the device driver for specifics. Examples of privilege_name
                would be "configuration_exclusive" for IOSXRDriver, or "configuration_private" for
                JunosDriver. You can also pass in a name of a configuration session such as
                "my-config-session" if you have registered a session using the
                "register_config_session" method of the EOSDriver or NXOSDriver.
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER CONFIG sent, not for the total
                of the configs being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        resolved_privilege_level, failed_when_contains = self._pre_send_configs(
            configs=configs,
            failed_when_contains=failed_when_contains,
            privilege_level=privilege_level,
        )

        if self._current_priv_level.name != resolved_privilege_level:
            await self.acquire_priv(desired_priv=resolved_privilege_level)

        responses = await super().send_commands(
            commands=configs,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            eager=eager,
            timeout_ops=timeout_ops,
        )

        if stop_on_failed and responses.failed:
            await self._abort_config()

        return self._post_send_configs(responses=responses)

    async def send_config(
        self,
        config: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        privilege_level: str = "",
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> Response:
        """
        Send configuration string

        Args:
            config: string configuration to send to the device, supports sending multi-line strings
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution; aborts configuration session if applicable (iosxr/junos or
                eos/nxos if using a configuration session)
            privilege_level: name of configuration privilege level/type to acquire; this is platform
                dependent, so check the device driver for specifics. Examples of privilege_name
                would be "configuration_exclusive" for IOSXRDriver, or "configuration_private" for
                JunosDriver. You can also pass in a name of a configuration session such as
                "my-config-session" if you have registered a session using the
                "register_config_session" method of the EOSDriver or NXOSDriver.
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER CONFIG sent, not for the total
                of the configs being sent!

        Returns:
            Response: Scrapli Response object

        Raises:
            N/A

        """
        split_config = self._pre_send_config(config=config)

        # now that we have a list of configs, just use send_configs to actually execute them
        multi_response = await self.send_configs(
            configs=split_config,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            privilege_level=privilege_level,
            eager=eager,
            timeout_ops=timeout_ops,
        )
        return self._post_send_config(config=config, multi_response=multi_response)

    async def send_configs_from_file(
        self,
        file: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        privilege_level: str = "",
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send configuration(s) from a file

        Args:
            file: string path to file
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution; aborts configuration session if applicable (iosxr/junos or
                eos/nxos if using a configuration session)
            privilege_level: name of configuration privilege level/type to acquire; this is platform
                dependent, so check the device driver for specifics. Examples of privilege_name
                would be "exclusive" for IOSXRDriver, "private" for JunosDriver. You can also pass
                in a name of a configuration session such as "session_mysession" if you have
                registered a session using the "register_config_session" method of the EOSDriver or
                NXOSDriver.
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER CONFIG sent, not for the total
                of the configs being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        configs = self._pre_send_from_file(file=file, caller="send_configs_from_file")

        return await self.send_configs(
            configs=configs,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            privilege_level=privilege_level,
            eager=eager,
            timeout_ops=timeout_ops,
        )
        </code>
    </pre>
</details>




## Classes

### AsyncNetworkDriver


```text
BaseDriver Object

BaseDriver is the root for all Scrapli driver classes. The synchronous and asyncio driver
base driver classes can be used to provide a semi-pexpect like experience over top of
whatever transport a user prefers. Generally, however, the base driver classes should not be
used directly. It is best to use the GenericDriver (or AsyncGenericDriver) or NetworkDriver
(or AsyncNetworkDriver) sub-classes of the base drivers.

Args:
    host: host ip/name to connect to
    port: port to connect to
    auth_username: username for authentication
    auth_private_key: path to private key for authentication
    auth_private_key_passphrase: passphrase for decrypting ssh key if necessary
    auth_password: password for authentication
    auth_strict_key: strict host checking or not
    auth_bypass: bypass "in channel" authentication -- only supported with telnet,
        asynctelnet, and system transport plugins
    timeout_socket: timeout for establishing socket/initial connection in seconds
    timeout_transport: timeout for ssh|telnet transport in seconds
    timeout_ops: timeout for ssh channel operations
    comms_prompt_pattern: raw string regex pattern -- preferably use `^` and `$` anchors!
        this is the single most important attribute here! if this does not match a prompt,
        scrapli will not work!
        IMPORTANT: regex search uses multi-line + case insensitive flags. multi-line allows
        for highly reliably matching for prompts however we do NOT strip trailing whitespace
        for each line, so be sure to add '\\s?' or similar if your device needs that. This
        should be mostly sorted for you if using network drivers (i.e. `IOSXEDriver`).
        Lastly, the case insensitive is just a convenience factor so i can be lazy.
    comms_return_char: character to use to send returns to host
    ssh_config_file: string to path for ssh config file, True to use default ssh config file
        or False to ignore default ssh config file
    ssh_known_hosts_file: string to path for ssh known hosts file, True to use default known
        file locations. Only applicable/needed if `auth_strict_key` is set to True
    on_init: callable that accepts the class instance as its only argument. this callable,
        if provided, is executed as the last step of object instantiation -- its purpose is
        primarily to provide a mechanism for scrapli community platforms to have an easy way
        to modify initialization arguments/object attributes without needing to create a
        class that extends the driver, instead allowing the community platforms to simply
        build from the GenericDriver or NetworkDriver classes, and pass this callable to do
        things such as appending to a username (looking at you RouterOS!!). Note that this
        is *always* a synchronous function (even for asyncio drivers)!
    on_open: callable that accepts the class instance as its only argument. this callable,
        if provided, is executed immediately after authentication is completed. Common use
        cases for this callable would be to disable paging or accept any kind of banner
        message that prompts a user upon connection
    on_close: callable that accepts the class instance as its only argument. this callable,
        if provided, is executed immediately prior to closing the underlying transport.
        Common use cases for this callable would be to save configurations prior to exiting,
        or to logout properly to free up vtys or similar
    transport: name of the transport plugin to use for the actual telnet/ssh/netconf
        connection. Available "core" transports are:
            - system
            - telnet
            - asynctelnet
            - ssh2
            - paramiko
            - asyncssh
        Please see relevant transport plugin section for details. Additionally third party
        transport plugins may be available.
    transport_options: dictionary of options to pass to selected transport class; see
        docs for given transport class for details of what to pass here
    channel_lock: True/False to lock the channel (threading.Lock/asyncio.Lock) during
        any channel operations, defaults to False
    channel_log: True/False or a string path to a file of where to write out channel logs --
        these are not "logs" in the normal logging module sense, but only the output that is
        read from the channel. In other words, the output of the channel log should look
        similar to what you would see as a human connecting to a device
    channel_log_mode: "write"|"append", all other values will raise ValueError,
        does what it sounds like it should by setting the channel log to the provided mode
    logging_uid: unique identifier (string) to associate to log messages; useful if you have
        multiple connections to the same device (i.e. one console, one ssh, or one to each
        supervisor module, etc.)

Returns:
    None

Raises:
    N/A
```

<details class="source">
    <summary>
        <span>Expand source code</span>
    </summary>
    <pre>
        <code class="python">
class AsyncNetworkDriver(AsyncGenericDriver, BaseNetworkDriver):
    def __init__(
        self,
        host: str,
        privilege_levels: Dict[str, PrivilegeLevel],
        default_desired_privilege_level: str,
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
        textfsm_platform: str = "",
        genie_platform: str = "",
    ):
        # ensure type for comms_prompt_pattern exists before setting it in the mixin
        self.comms_prompt_pattern: str

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
        )

        self.auth_secondary = auth_secondary
        self.failed_when_contains = failed_when_contains or []
        self.textfsm_platform = textfsm_platform
        self.genie_platform = genie_platform

        self.privilege_levels = privilege_levels
        self.default_desired_privilege_level = default_desired_privilege_level
        self._priv_graph = defaultdict(set)
        self.update_privilege_levels()

    async def _escalate(self, escalate_priv: PrivilegeLevel) -> None:
        """
        Escalate to the next privilege level up

        Args:
            escalate_priv: privilege level to escalate to

        Returns:
            None

        Raises:
            N/A

        """
        self._pre_escalate(escalate_priv=escalate_priv)

        if escalate_priv.escalate_auth is True and self.auth_secondary:
            await super().send_interactive(
                interact_events=[
                    (escalate_priv.escalate, escalate_priv.escalate_prompt, False),
                    (self.auth_secondary, escalate_priv.pattern, True),
                ],
            )
        else:
            await self.channel.send_input(channel_input=escalate_priv.escalate)

    async def _deescalate(self, current_priv: PrivilegeLevel) -> None:
        """
        Deescalate to the next privilege level down

        Args:
            current_priv: current privilege level

        Returns:
            None

        Raises:
            N/A

        """
        await self.channel.send_input(channel_input=current_priv.deescalate)

    async def acquire_priv(self, desired_priv: str) -> None:
        """
        Acquire desired priv level

        Args:
            desired_priv: string name of desired privilege level see
                `scrapli.driver.<driver_category.device_type>.driver` for levels

        Returns:
            None

        Raises:
            ScrapliPrivilegeError: if desired_priv cannot be attained

        """
        self._validate_privilege_level_name(privilege_level_name=desired_priv)

        privilege_change_count = 0

        while True:
            current_prompt = await self.channel.get_prompt()
            privilege_action, target_priv = self._process_acquire_priv(
                destination_priv=desired_priv,
                current_prompt=current_prompt,
            )

            if privilege_action == PrivilegeAction.NO_ACTION:
                self._current_priv_level = target_priv
                return
            if privilege_action == PrivilegeAction.DEESCALATE:
                await self._deescalate(current_priv=target_priv)
            if privilege_action == PrivilegeAction.ESCALATE:
                await self._escalate(escalate_priv=target_priv)

            privilege_change_count += 1
            if privilege_change_count > len(self.privilege_levels) * 2:
                msg = f"Failed to acquire requested privilege level {desired_priv}"
                raise ScrapliPrivilegeError(msg)

    async def send_command(
        self,
        command: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        timeout_ops: Optional[float] = None,
    ) -> Response:
        """
        Send a command

        Super method will raise TypeError if anything but a string is passed here!

        Args:
            command: string to send to device in privilege exec mode
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed

        Returns:
            Response: Scrapli Response object

        Raises:
            N/A

        """
        if self._current_priv_level.name != self.default_desired_privilege_level:
            await self.acquire_priv(desired_priv=self.default_desired_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        response = await super().send_command(
            command=command,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            timeout_ops=timeout_ops,
        )
        self._update_response(response)

        return response

    async def send_commands(
        self,
        commands: List[str],
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send multiple commands

        Super method will raise TypeError if anything but a list of strings is passed here!

        Args:
            commands: list of strings to send to device in privilege exec mode
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER COMMAND sent, not for the total
                of the commands being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        if self._current_priv_level.name != self.default_desired_privilege_level:
            await self.acquire_priv(desired_priv=self.default_desired_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        responses = await super().send_commands(
            commands=commands,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            eager=eager,
            timeout_ops=timeout_ops,
        )

        for response in responses:
            self._update_response(response=response)

        return responses

    async def send_commands_from_file(
        self,
        file: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send command(s) from file

        Args:
            file: string path to file
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER COMMAND sent, not for the total
                of the commands being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        if self._current_priv_level.name != self.default_desired_privilege_level:
            await self.acquire_priv(desired_priv=self.default_desired_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        return await super().send_commands_from_file(
            file=file,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            eager=eager,
            timeout_ops=timeout_ops,
        )

    async def send_interactive(
        self,
        interact_events: Union[List[Tuple[str, str]], List[Tuple[str, str, bool]]],
        *,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        privilege_level: str = "",
        timeout_ops: Optional[float] = None,
    ) -> Response:
        """
        Interact with a device with changing prompts per input.

        Used to interact with devices where prompts change per input, and where inputs may be hidden
        such as in the case of a password input. This can be used to respond to challenges from
        devices such as the confirmation for the command "clear logging" on IOSXE devices for
        example. You may have as many elements in the "interact_events" list as needed, and each
        element of that list should be a tuple of two or three elements. The first element is always
        the input to send as a string, the second should be the expected response as a string, and
        the optional third a bool for whether or not the input is "hidden" (i.e. password input)

        An example where we need this sort of capability:

        '''
        3560CX#copy flash: scp:
        Source filename []? test1.txt
        Address or name of remote host []? 172.31.254.100
        Destination username [carl]?
        Writing test1.txt
        Password:

        Password:
         Sink: C0644 639 test1.txt
        !
        639 bytes copied in 12.066 secs (53 bytes/sec)
        3560CX#
        '''

        To accomplish this we can use the following:

        '''
        interact = conn.channel.send_inputs_interact(
            [
                ("copy flash: scp:", "Source filename []?", False),
                ("test1.txt", "Address or name of remote host []?", False),
                ("172.31.254.100", "Destination username [carl]?", False),
                ("carl", "Password:", False),
                ("super_secure_password", prompt, True),
            ]
        )
        '''

        If we needed to deal with more prompts we could simply continue adding tuples to the list of
        interact "events".

        Args:
            interact_events: list of tuples containing the "interactions" with the device
                each list element must have an input and an expected response, and may have an
                optional bool for the third and final element -- the optional bool specifies if the
                input that is sent to the device is "hidden" (ex: password), if the hidden param is
                not provided it is assumed the input is "normal" (not hidden)
            failed_when_contains: list of strings that, if present in final output, represent a
                failed command/interaction
            privilege_level: name of the privilege level to operate in
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER COMMAND sent, not for the total
                of the commands being sent!

        Returns:
            Response: scrapli Response object

        Raises:
            N/A

        """
        if privilege_level:
            self._validate_privilege_level_name(privilege_level_name=privilege_level)
            resolved_privilege_level = privilege_level
        else:
            resolved_privilege_level = self.default_desired_privilege_level

        if self._current_priv_level.name != resolved_privilege_level:
            await self.acquire_priv(desired_priv=resolved_privilege_level)

        if failed_when_contains is None:
            failed_when_contains = self.failed_when_contains

        # type hint is due to the TimeoutModifier wrapper returning `Any` so that we dont anger the
        # asyncio parts (which will get an awaitable not a Response returned)
        response: Response = await super().send_interactive(
            interact_events=interact_events,
            failed_when_contains=failed_when_contains,
            timeout_ops=timeout_ops,
        )
        self._update_response(response=response)

        return response

    async def _abort_config(self) -> None:
        """
        Abort a configuration operation/session if applicable (for config sessions like junos/iosxr)

        Args:
            N/A

        Returns:
            None

        Raises:
            N/A

        """

    async def send_configs(
        self,
        configs: List[str],
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        privilege_level: str = "",
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send configuration(s)

        Args:
            configs: list of strings to send to device in config mode
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution; aborts configuration session if applicable (iosxr/junos or
                eos/nxos if using a configuration session)
            privilege_level: name of configuration privilege level/type to acquire; this is platform
                dependent, so check the device driver for specifics. Examples of privilege_name
                would be "configuration_exclusive" for IOSXRDriver, or "configuration_private" for
                JunosDriver. You can also pass in a name of a configuration session such as
                "my-config-session" if you have registered a session using the
                "register_config_session" method of the EOSDriver or NXOSDriver.
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER CONFIG sent, not for the total
                of the configs being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        resolved_privilege_level, failed_when_contains = self._pre_send_configs(
            configs=configs,
            failed_when_contains=failed_when_contains,
            privilege_level=privilege_level,
        )

        if self._current_priv_level.name != resolved_privilege_level:
            await self.acquire_priv(desired_priv=resolved_privilege_level)

        responses = await super().send_commands(
            commands=configs,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            eager=eager,
            timeout_ops=timeout_ops,
        )

        if stop_on_failed and responses.failed:
            await self._abort_config()

        return self._post_send_configs(responses=responses)

    async def send_config(
        self,
        config: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        privilege_level: str = "",
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> Response:
        """
        Send configuration string

        Args:
            config: string configuration to send to the device, supports sending multi-line strings
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution; aborts configuration session if applicable (iosxr/junos or
                eos/nxos if using a configuration session)
            privilege_level: name of configuration privilege level/type to acquire; this is platform
                dependent, so check the device driver for specifics. Examples of privilege_name
                would be "configuration_exclusive" for IOSXRDriver, or "configuration_private" for
                JunosDriver. You can also pass in a name of a configuration session such as
                "my-config-session" if you have registered a session using the
                "register_config_session" method of the EOSDriver or NXOSDriver.
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER CONFIG sent, not for the total
                of the configs being sent!

        Returns:
            Response: Scrapli Response object

        Raises:
            N/A

        """
        split_config = self._pre_send_config(config=config)

        # now that we have a list of configs, just use send_configs to actually execute them
        multi_response = await self.send_configs(
            configs=split_config,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            privilege_level=privilege_level,
            eager=eager,
            timeout_ops=timeout_ops,
        )
        return self._post_send_config(config=config, multi_response=multi_response)

    async def send_configs_from_file(
        self,
        file: str,
        *,
        strip_prompt: bool = True,
        failed_when_contains: Optional[Union[str, List[str]]] = None,
        stop_on_failed: bool = False,
        privilege_level: str = "",
        eager: bool = False,
        timeout_ops: Optional[float] = None,
    ) -> MultiResponse:
        """
        Send configuration(s) from a file

        Args:
            file: string path to file
            strip_prompt: True/False strip prompt from returned output
            failed_when_contains: string or list of strings indicating failure if found in response
            stop_on_failed: True/False stop executing commands if a command fails, returns results
                as of current execution; aborts configuration session if applicable (iosxr/junos or
                eos/nxos if using a configuration session)
            privilege_level: name of configuration privilege level/type to acquire; this is platform
                dependent, so check the device driver for specifics. Examples of privilege_name
                would be "exclusive" for IOSXRDriver, "private" for JunosDriver. You can also pass
                in a name of a configuration session such as "session_mysession" if you have
                registered a session using the "register_config_session" method of the EOSDriver or
                NXOSDriver.
            eager: if eager is True we do not read until prompt is seen at each command sent to the
                channel. Do *not* use this unless you know what you are doing as it is possible that
                it can make scrapli less reliable!
            timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
                the duration of the operation, value is reset to initial value after operation is
                completed. Note that this is the timeout value PER CONFIG sent, not for the total
                of the configs being sent!

        Returns:
            MultiResponse: Scrapli MultiResponse object

        Raises:
            N/A

        """
        configs = self._pre_send_from_file(file=file, caller="send_configs_from_file")

        return await self.send_configs(
            configs=configs,
            strip_prompt=strip_prompt,
            failed_when_contains=failed_when_contains,
            stop_on_failed=stop_on_failed,
            privilege_level=privilege_level,
            eager=eager,
            timeout_ops=timeout_ops,
        )
        </code>
    </pre>
</details>


#### Ancestors (in MRO)
- scrapli.driver.generic.async_driver.AsyncGenericDriver
- scrapli.driver.base.async_driver.AsyncDriver
- scrapli.driver.base.base_driver.BaseDriver
- scrapli.driver.generic.base_driver.BaseGenericDriver
- scrapli.driver.network.base_driver.BaseNetworkDriver
#### Descendants
- scrapli.driver.core.arista_eos.async_driver.AsyncEOSDriver
- scrapli.driver.core.cisco_iosxe.async_driver.AsyncIOSXEDriver
- scrapli.driver.core.cisco_iosxr.async_driver.AsyncIOSXRDriver
- scrapli.driver.core.cisco_nxos.async_driver.AsyncNXOSDriver
- scrapli.driver.core.juniper_junos.async_driver.AsyncJunosDriver
- scrapli.factory.AsyncScrapli
#### Class variables

    
`auth_secondary: str`




    
`failed_when_contains: List[str]`




    
`genie_platform: str`




    
`logger: logging.LoggerAdapter`




    
`privilege_levels: Dict[str, scrapli.driver.network.base_driver.PrivilegeLevel]`




    
`textfsm_platform: str`



#### Methods

    

##### acquire_priv
`acquire_priv(self, desired_priv: str) ‑> NoneType`

```text
Acquire desired priv level

Args:
    desired_priv: string name of desired privilege level see
        `scrapli.driver.<driver_category.device_type>.driver` for levels

Returns:
    None

Raises:
    ScrapliPrivilegeError: if desired_priv cannot be attained
```



    

##### send_command
`send_command(self, command: str, *, strip_prompt: bool = True, failed_when_contains: Union[str, List[str], NoneType] = None, timeout_ops: Optional[float] = None) ‑> scrapli.response.Response`

```text
Send a command

Super method will raise TypeError if anything but a string is passed here!

Args:
    command: string to send to device in privilege exec mode
    strip_prompt: True/False strip prompt from returned output
    failed_when_contains: string or list of strings indicating failure if found in response
    timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
        the duration of the operation, value is reset to initial value after operation is
        completed

Returns:
    Response: Scrapli Response object

Raises:
    N/A
```



    

##### send_commands
`send_commands(self, commands: List[str], *, strip_prompt: bool = True, failed_when_contains: Union[str, List[str], NoneType] = None, stop_on_failed: bool = False, eager: bool = False, timeout_ops: Optional[float] = None) ‑> scrapli.response.MultiResponse`

```text
Send multiple commands

Super method will raise TypeError if anything but a list of strings is passed here!

Args:
    commands: list of strings to send to device in privilege exec mode
    strip_prompt: True/False strip prompt from returned output
    failed_when_contains: string or list of strings indicating failure if found in response
    stop_on_failed: True/False stop executing commands if a command fails, returns results
        as of current execution
    eager: if eager is True we do not read until prompt is seen at each command sent to the
        channel. Do *not* use this unless you know what you are doing as it is possible that
        it can make scrapli less reliable!
    timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
        the duration of the operation, value is reset to initial value after operation is
        completed. Note that this is the timeout value PER COMMAND sent, not for the total
        of the commands being sent!

Returns:
    MultiResponse: Scrapli MultiResponse object

Raises:
    N/A
```



    

##### send_config
`send_config(self, config: str, *, strip_prompt: bool = True, failed_when_contains: Union[str, List[str], NoneType] = None, stop_on_failed: bool = False, privilege_level: str = '', eager: bool = False, timeout_ops: Optional[float] = None) ‑> scrapli.response.Response`

```text
Send configuration string

Args:
    config: string configuration to send to the device, supports sending multi-line strings
    strip_prompt: True/False strip prompt from returned output
    failed_when_contains: string or list of strings indicating failure if found in response
    stop_on_failed: True/False stop executing commands if a command fails, returns results
        as of current execution; aborts configuration session if applicable (iosxr/junos or
        eos/nxos if using a configuration session)
    privilege_level: name of configuration privilege level/type to acquire; this is platform
        dependent, so check the device driver for specifics. Examples of privilege_name
        would be "configuration_exclusive" for IOSXRDriver, or "configuration_private" for
        JunosDriver. You can also pass in a name of a configuration session such as
        "my-config-session" if you have registered a session using the
        "register_config_session" method of the EOSDriver or NXOSDriver.
    eager: if eager is True we do not read until prompt is seen at each command sent to the
        channel. Do *not* use this unless you know what you are doing as it is possible that
        it can make scrapli less reliable!
    timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
        the duration of the operation, value is reset to initial value after operation is
        completed. Note that this is the timeout value PER CONFIG sent, not for the total
        of the configs being sent!

Returns:
    Response: Scrapli Response object

Raises:
    N/A
```



    

##### send_configs
`send_configs(self, configs: List[str], *, strip_prompt: bool = True, failed_when_contains: Union[str, List[str], NoneType] = None, stop_on_failed: bool = False, privilege_level: str = '', eager: bool = False, timeout_ops: Optional[float] = None) ‑> scrapli.response.MultiResponse`

```text
Send configuration(s)

Args:
    configs: list of strings to send to device in config mode
    strip_prompt: True/False strip prompt from returned output
    failed_when_contains: string or list of strings indicating failure if found in response
    stop_on_failed: True/False stop executing commands if a command fails, returns results
        as of current execution; aborts configuration session if applicable (iosxr/junos or
        eos/nxos if using a configuration session)
    privilege_level: name of configuration privilege level/type to acquire; this is platform
        dependent, so check the device driver for specifics. Examples of privilege_name
        would be "configuration_exclusive" for IOSXRDriver, or "configuration_private" for
        JunosDriver. You can also pass in a name of a configuration session such as
        "my-config-session" if you have registered a session using the
        "register_config_session" method of the EOSDriver or NXOSDriver.
    eager: if eager is True we do not read until prompt is seen at each command sent to the
        channel. Do *not* use this unless you know what you are doing as it is possible that
        it can make scrapli less reliable!
    timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
        the duration of the operation, value is reset to initial value after operation is
        completed. Note that this is the timeout value PER CONFIG sent, not for the total
        of the configs being sent!

Returns:
    MultiResponse: Scrapli MultiResponse object

Raises:
    N/A
```



    

##### send_configs_from_file
`send_configs_from_file(self, file: str, *, strip_prompt: bool = True, failed_when_contains: Union[str, List[str], NoneType] = None, stop_on_failed: bool = False, privilege_level: str = '', eager: bool = False, timeout_ops: Optional[float] = None) ‑> scrapli.response.MultiResponse`

```text
Send configuration(s) from a file

Args:
    file: string path to file
    strip_prompt: True/False strip prompt from returned output
    failed_when_contains: string or list of strings indicating failure if found in response
    stop_on_failed: True/False stop executing commands if a command fails, returns results
        as of current execution; aborts configuration session if applicable (iosxr/junos or
        eos/nxos if using a configuration session)
    privilege_level: name of configuration privilege level/type to acquire; this is platform
        dependent, so check the device driver for specifics. Examples of privilege_name
        would be "exclusive" for IOSXRDriver, "private" for JunosDriver. You can also pass
        in a name of a configuration session such as "session_mysession" if you have
        registered a session using the "register_config_session" method of the EOSDriver or
        NXOSDriver.
    eager: if eager is True we do not read until prompt is seen at each command sent to the
        channel. Do *not* use this unless you know what you are doing as it is possible that
        it can make scrapli less reliable!
    timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
        the duration of the operation, value is reset to initial value after operation is
        completed. Note that this is the timeout value PER CONFIG sent, not for the total
        of the configs being sent!

Returns:
    MultiResponse: Scrapli MultiResponse object

Raises:
    N/A
```



    

##### send_interactive
`send_interactive(self, interact_events: Union[List[Tuple[str, str]], List[Tuple[str, str, bool]]], *, failed_when_contains: Union[str, List[str], NoneType] = None, privilege_level: str = '', timeout_ops: Optional[float] = None) ‑> scrapli.response.Response`

```text
Interact with a device with changing prompts per input.

Used to interact with devices where prompts change per input, and where inputs may be hidden
such as in the case of a password input. This can be used to respond to challenges from
devices such as the confirmation for the command "clear logging" on IOSXE devices for
example. You may have as many elements in the "interact_events" list as needed, and each
element of that list should be a tuple of two or three elements. The first element is always
the input to send as a string, the second should be the expected response as a string, and
the optional third a bool for whether or not the input is "hidden" (i.e. password input)

An example where we need this sort of capability:

'''
3560CX#copy flash: scp:
Source filename []? test1.txt
Address or name of remote host []? 172.31.254.100
Destination username [carl]?
Writing test1.txt
Password:

Password:
 Sink: C0644 639 test1.txt
!
639 bytes copied in 12.066 secs (53 bytes/sec)
3560CX#
'''

To accomplish this we can use the following:

'''
interact = conn.channel.send_inputs_interact(
    [
        ("copy flash: scp:", "Source filename []?", False),
        ("test1.txt", "Address or name of remote host []?", False),
        ("172.31.254.100", "Destination username [carl]?", False),
        ("carl", "Password:", False),
        ("super_secure_password", prompt, True),
    ]
)
'''

If we needed to deal with more prompts we could simply continue adding tuples to the list of
interact "events".

Args:
    interact_events: list of tuples containing the "interactions" with the device
        each list element must have an input and an expected response, and may have an
        optional bool for the third and final element -- the optional bool specifies if the
        input that is sent to the device is "hidden" (ex: password), if the hidden param is
        not provided it is assumed the input is "normal" (not hidden)
    failed_when_contains: list of strings that, if present in final output, represent a
        failed command/interaction
    privilege_level: name of the privilege level to operate in
    timeout_ops: timeout ops value for this operation; only sets the timeout_ops value for
        the duration of the operation, value is reset to initial value after operation is
        completed. Note that this is the timeout value PER COMMAND sent, not for the total
        of the commands being sent!

Returns:
    Response: scrapli Response object

Raises:
    N/A
```