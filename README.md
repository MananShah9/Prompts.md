import cv2
import mediapipe as mp
import pyautogui
import time
import platform

# --- Configuration ---
# Adjust these thresholds based on your testing.
LEFT_THRESHOLD = -0.15  # Change in x-coordinate of wrist, normalized
RIGHT_THRESHOLD = 0.15
UP_THRESHOLD = -0.15  # Change in y-coordinate of wrist, normalized
DOWN_THRESHOLD = 0.1  # Change in y-coordinate of wrist
DELAY = 0.1 # delay to prevent multiple inputs

# --- Setup ---
mp_drawing = mp.solutions.drawing_utils
mp_pose = mp.solutions.pose

# Ensure consistent behavior across different platforms
pyautogui.PAUSE = 0.01  # Small pause between pyautogui actions
pyautogui.FAILSAFE = True  # Move mouse to top-left corner to stop

# --- Helper Functions ---

def calculate_normalized_change(previous_landmark, current_landmark):
    """Calculates the normalized change in x and y coordinates."""
    if previous_landmark is None or current_landmark is None:
        return 0, 0
    return (current_landmark.x - previous_landmark.x,
            current_landmark.y - previous_landmark.y)

def detect_movement(previous_left_wrist, current_left_wrist,
                    previous_right_wrist, current_right_wrist):
    """Detects movement based on wrist position changes."""
    left_x_change, left_y_change = calculate_normalized_change(
        previous_left_wrist, current_left_wrist
    )
    right_x_change, right_y_change = calculate_normalized_change(
        previous_right_wrist, current_right_wrist
    )

    action = "None"

    if left_x_change < LEFT_THRESHOLD or right_x_change < LEFT_THRESHOLD:
        action = "Left"
    elif left_x_change > RIGHT_THRESHOLD or right_x_change > RIGHT_THRESHOLD:
        action = "Right"
    elif left_y_change < UP_THRESHOLD or right_y_change < UP_THRESHOLD:
        action = "Up"
    elif left_y_change > DOWN_THRESHOLD or right_y_change > DOWN_THRESHOLD:
        action = "Down"
    return action



def main():
    cap = cv2.VideoCapture(0)  # 0 is usually the default webcam
    if not cap.isOpened():
        print("Error: Cannot open webcam.")
        return

    previous_left_wrist = None
    previous_right_wrist = None
    cooldown_end_time = 0

    with mp_pose.Pose(min_detection_confidence=0.5, min_tracking_confidence=0.5) as pose:
        while cap.isOpened():
            success, image = cap.read()
            if not success:
                print("Ignoring empty camera frame.")
                continue

            # Flip the image horizontally for a later selfie-view display
            image = cv2.flip(image, 1)
            # Convert the BGR image to RGB.
            image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)

            # To improve performance, optionally mark the image as not writeable
            image.flags.writeable = False
            results = pose.process(image)

            # Draw the pose annotations on the image.
            image.flags.writeable = True
            image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)

            if results.pose_landmarks:
                mp_drawing.draw_landmarks(
                    image, results.pose_landmarks, mp_pose.POSE_CONNECTIONS)

                # Get landmark positions
                landmarks = results.pose_landmarks.landmark
                current_left_wrist = landmarks[mp_pose.PoseLandmark.LEFT_WRIST]
                current_right_wrist = landmarks[mp_pose.PoseLandmark.RIGHT_WRIST]

                if time.time() > cooldown_end_time:
                  action = detect_movement(previous_left_wrist, current_left_wrist,
                                          previous_right_wrist, current_right_wrist)

                  if action != "None":
                      print(f"Action: {action}")
                      if action == "Left":
                          pyautogui.press('left')
                      elif action == "Right":
                          pyautogui.press('right')
                      elif action == "Up":
                          pyautogui.press('up')
                      elif action == "Down":
                          pyautogui.press('down')
                      cooldown_end_time = time.time() + DELAY  # Set cooldown

                previous_left_wrist = current_left_wrist
                previous_right_wrist = current_right_wrist


            cv2.imshow('MediaPipe Pose', image)
            if cv2.waitKey(5) & 0xFF == 27:  # Press ESC to exit
                break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
