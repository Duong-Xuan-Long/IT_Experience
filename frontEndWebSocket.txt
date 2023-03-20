import useFilterNotification from 'hooks/filter/useFilterNotification';
import useAuth from 'hooks/useAuth';
import useNotification from 'hooks/useNotification';
import useRefresh from 'hooks/useRefresh';
import {
  createContext,
  Dispatch,
  DispatchWithoutAction,
  SetStateAction,
  useCallback,
  useEffect,
  useRef,
  useState,
} from 'react';
import { useTranslation } from 'react-i18next';
import Notification from 'services/notification';
import SockJS from 'sockjs-client';
import { Client, Message, over } from 'stompjs';
import { NotificationType } from 'types/notification';
import type { FCC } from 'types/react';
// import { adminSocketURL, clientSocketURL } from 'utils/config';
import { clientSocketURL } from 'utils/config';

interface State {
  unReadCount: number;
  setUnReadCount: Dispatch<SetStateAction<number>>;
  notifications: NotificationType[];
  setNotifications: Dispatch<SetStateAction<NotificationType[]>>;
  loadingScreen: boolean;
  setLoadingScreen: Dispatch<SetStateAction<boolean>>;
  refetch: DispatchWithoutAction;
}

export interface SocketNotificationValue extends State {}

const SocketNotificationContext = createContext<SocketNotificationValue | null>(
  null
);

if (process.env.NODE_ENV === 'development') {
  SocketNotificationContext.displayName = 'SocketNotificationContext';
}

const SocketNotificationProvider: FCC = (props) => {
  const { children } = props;
  const [refresh, refetch] = useRefresh();
  const { user } = useAuth();
  // const stompAdmin = useRef<null | Client>(null);
  // const socketAdmin = useRef<null | WebSocket>(null);
  const stompMember = useRef<null | Client>(null);
  const socketMember = useRef<null | WebSocket>(null);
  const setNotification = useNotification();
  const { t } = useTranslation();
  const [unReadCount, setUnReadCount] = useState<number>(0);
  const [notifications, setNotifications] = useState<NotificationType[]>([]);
  const [loadingScreen, setLoadingScreen] = useState<boolean>(false);
  const { filterNotification } = useFilterNotification();

  useEffect(() => {
    Notification.getUnReadNotification({ userCode: user?.userCode || '' }).then(
      (response) => {
        if (response.data) {
          setUnReadCount(response.data);
        }
      }
    );
    // eslint-disable-next-line
  }, [user]);

  useEffect(() => {
    setLoadingScreen(true);
    Notification.getAllNotification({
      ...filterNotification,
      userCode: user?.userCode || '',
    })
      .then((response) => {
        if (response.data && response.data?.length > 0) {
          setNotifications(response.data);
        }
      })
      .catch((error) => {})
      .finally(() => {
        setLoadingScreen(false);
      });
    // eslint-disable-next-line
  }, [user, refresh]);

  // const onConnectedAdmin = useCallback(() => {
  //   if (stompAdmin.current && stompAdmin.current.connected && user) {
  //     stompAdmin.current.subscribe(
  //       `/user/${user.userCode}/private`,
  //       onGetMessage
  //     );
  //     stompAdmin.current.send(
  //       '/private-message',
  //       {},
  //       JSON.stringify({ messageContent: `${user.userCode}` })
  //     );
  //   }
  //   // eslint-disable-next-line
  // }, [user]);

  const onConnectedMember = useCallback(() => {
    if (stompMember.current && stompMember.current.connected && user) {
      stompMember.current.subscribe(
        `/user/${user.userCode}/private`,
        onGetMessage
      );
      stompMember.current.send(
        '/private-message',
        {},
        JSON.stringify({ messageContent: `${user.userCode}` })
      );
    }
    // eslint-disable-next-line
  }, [user]);

  // //Admin
  // useEffect(() => {
  //   // Init socket
  //   if (
  //     user &&
  //     (!socketAdmin.current ||
  //       (socketAdmin.current && socketAdmin.current.readyState !== 1))
  //   ) {
  //     socketAdmin.current = new SockJS(adminSocketURL as string);
  //   }

  //   // Close socket when logout
  //   if (!user && socketAdmin.current && socketAdmin.current.readyState === 1) {
  //     socketAdmin.current.close();
  //   }
  //   // Connect stomp client
  //   if (user && socketAdmin.current) {
  //     stompAdmin.current = over(socketAdmin.current);
  //     stompAdmin.current.connect({}, onConnectedAdmin, (err) => {
  //       console.log(err);
  //     });
  //   }
  //   // Logout
  //   if (!user && stompAdmin.current && stompAdmin.current.connected) {
  //     stompAdmin.current.disconnect(() => {});
  //   }
  // }, [user, onConnectedAdmin]);

  //Member
  useEffect(() => {
    // Init socket
    if (
      user &&
      (!socketMember.current ||
        (socketMember.current && socketMember.current.readyState !== 1))
    ) {
      socketMember.current = new SockJS(clientSocketURL as string);
    }

    // Close socket when logout
    if (
      !user &&
      socketMember.current &&
      socketMember.current.readyState === 1
    ) {
      socketMember.current.close();
    }
    // Connect stomp client
    if (user && socketMember.current) {
      stompMember.current = over(socketMember.current);
      stompMember.current.connect({}, onConnectedMember, (err) => {
        console.log(err);
      });
    }
    // Logout
    if (!user && stompMember.current && stompMember.current.connected) {
      stompMember.current.disconnect(() => {});
    }
  }, [user, onConnectedMember]);

  const onGetMessage = (payload: Message) => {
    setNotification({
      message: t('label.notification.newNotify'),
      severity: 'warning',
    });
    setUnReadCount((prev) => prev + 1);
    refetch();
  };

  return (
    <SocketNotificationContext.Provider
      value={{
        unReadCount,
        setUnReadCount,
        notifications,
        setNotifications,
        loadingScreen,
        setLoadingScreen,
        refetch,
      }}
    >
      {children}
    </SocketNotificationContext.Provider>
  );
};

const AuthConsumer = SocketNotificationContext.Consumer;
export {
  SocketNotificationContext as default,
  SocketNotificationProvider,
  AuthConsumer,
};
