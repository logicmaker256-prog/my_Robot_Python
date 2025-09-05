import sys
import termios
import tty
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

msg = """
Reading from the keyboard and Publishing to /keyboard_vel!
---------------------------
Moving around:
        w
   a    s    d
        x

w/x : increase/decrease linear velocity
a/d : increase/decrease angular velocity
space key, s : force stop
CTRL-C to quit
"""

move_bindings = {
    'w': (1, 0),
    'x': (-1, 0),
    'a': (0, 1),
    'd': (0, -1),
}

speed_bindings = {
    'q': (1.1, 1.1),
    'z': (0.9, 0.9),
    'e': (1.1, 1.0),
    'c': (0.9, 1.0),
    'r': (1.0, 1.1),
    'v': (1.0, 0.9),
}

def get_key(settings):
    tty.setraw(sys.stdin.fileno())
    key = sys.stdin.read(1)
    termios.tcsetattr(sys.stdin, termios.TCSADRAIN, settings)
    return key

class TeleopKeyboard(Node):
    def __init__(self):
        super().__init__('teleop_keyboard')
        self.pub = self.create_publisher(Twist, '/keyboard_vel', 10)
        self.settings = termios.tcgetattr(sys.stdin)

        self.speed = 0.5
        self.turn = 1.0

    def run(self):
        print(msg)
        while rclpy.ok():
            key = get_key(self.settings)
            twist = Twist()

            if key in move_bindings.keys():
                x = move_bindings[key][0]
                th = move_bindings[key][1]
                twist.linear.x = x * self.speed
                twist.angular.z = th * self.turn
            elif key in speed_bindings.keys():
                self.speed *= speed_bindings[key][0]
                self.turn *= speed_bindings[key][1]
                print(f"current speed {self.speed:.2f}, turn {self.turn:.2f}")
                continue
            elif key == ' ' or key == 's':
                twist.linear.x = 0.0
                twist.angular.z = 0.0
            elif key == '\x03':  # Ctrl-C
                break
            else:
                continue

            self.pub.publish(twist)

def main(args=None):
    rclpy.init(args=args)
    node = TeleopKeyboard()
    try:
        node.run()
    except Exception as e:
        print(e)
    finally:
        twist = Twist()
        node.pub.publish(twist)
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()