import cv2
import mediapipe as mp
import pyautogui
import math
import time
import numpy as np  # Import numpy

class PoseController:
    def __init__(self, frame_width=640, frame_height=480):
        self.frame_width = frame_width
        self.frame_height = frame_height

        # --- MediaPipe Pose Setup ---
        self.mp_pose = mp.solutions.pose
        self.pose = self.mp_pose.Pose(
            min_detection_confidence=0.7,
            min_tracking_confidence=0.7,
            model_complexity=1  # Adjust for performance
        )
        self.mp_drawing = mp.solutions.drawing_utils

        # --- Webcam Setup ---
        self.cap = cv2.VideoCapture(0)
        if not self.cap.isOpened():
            raise Exception("Cannot open webcam")
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, self.frame_width)
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, self.frame_height)

        # --- Calibration Data ---
        self.neutral_nose_x = None
        self.left_threshold = None
        self.right_threshold = None
        self.neutral_shoulder_y = None

        # --- Gesture Parameters ---
        self.jump_threshold = 25  # Adjusted threshold
        self.duck_threshold = 15   # Adjusted threshold
        self.horizontal_deadzone = 0.1  # 10% of frame width
        self.jump_debounce_time = 0.4 # seconds
        self.duck_debounce_time = 0.4  # seconds
        self.last_jump_time = 0
        self.last_duck_time = 0
        self.smoothing_factor = 0.5  # For moving average

        # --- State Variables ---
        self.smoothed_nose_x = None
        self.smoothed_shoulder_y = None
        self.current_action = "Neutral"

    def calibrate(self):
        print("Calibration: Stand in a neutral pose facing the camera...")
        time.sleep(3)  # Give the user time to get ready

        ret, frame = self.cap.read()
        if not ret:
            raise Exception("Failed to read frame during calibration")

        frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        results = self.pose.process(frame_rgb)

        if results.pose_landmarks:
            landmarks = results.pose_landmarks.landmark
            if (landmarks[self.mp_pose.PoseLandmark.NOSE].visibility > 0.9 and
                landmarks[self.mp_pose.PoseLandmark.LEFT_SHOULDER].visibility > 0.9 and
                landmarks[self.mp_pose.PoseLandmark.RIGHT_SHOULDER].visibility > 0.9):

                self.neutral_nose_x = landmarks[self.mp_pose.PoseLandmark.NOSE].x
                self.left_threshold = self.neutral_nose_x - self.horizontal_deadzone
                self.right_threshold = self.neutral_nose_x + self.horizontal_deadzone

                self.neutral_shoulder_y = (landmarks[self.mp_pose.PoseLandmark.LEFT_SHOULDER].y +
                                           landmarks[self.mp_pose.PoseLandmark.RIGHT_SHOULDER].y) / 2

                self.smoothed_nose_x = self.neutral_nose_x  # Initialize smoothed values
                self.smoothed_shoulder_y = self.neutral_shoulder_y

                print("Calibration successful!")
                print(f"  Neutral Nose X: {self.neutral_nose_x:.2f}")
                print(f"  Left Threshold: {self.left_threshold:.2f}")
                print(f"  Right Threshold: {self.right_threshold:.2f}")
                print(f"  Neutral Shoulder Y: {self.neutral_shoulder_y:.2f}")
            else:
                raise Exception("Calibration failed: Landmarks not visible. Ensure good lighting and clear view.")
        else:
            raise Exception("Calibration failed: No pose detected.")


    def process_frame(self, frame):
        frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        results = self.pose.process(frame_rgb)

        if results.pose_landmarks:
            landmarks = results.pose_landmarks.landmark
            self.update_action(landmarks)
            self.mp_drawing.draw_landmarks(frame, results.pose_landmarks, self.mp_pose.POSE_CONNECTIONS)

        # Display the current action on the frame
        cv2.putText(frame, f"Action: {self.current_action}", (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
        return frame

    def update_action(self, landmarks):
        if (landmarks[self.mp_pose.PoseLandmark.NOSE].visibility > 0.7 and
            landmarks[self.mp_pose.PoseLandmark.LEFT_SHOULDER].visibility > 0.7 and
            landmarks[self.mp_pose.PoseLandmark.RIGHT_SHOULDER].visibility > 0.7):

            # --- Left/Right (Nose Position) ---
            nose_x = landmarks[self.mp_pose.PoseLandmark.NOSE].x
            self.smoothed_nose_x = (self.smoothing_factor * nose_x) + ((1 - self.smoothing_factor) * self.smoothed_nose_x)

            if self.smoothed_nose_x < self.left_threshold:
                self.current_action = "Left"
                pyautogui.press('left')
            elif self.smoothed_nose_x > self.right_threshold:
                self.current_action = "Right"
                pyautogui.press('right')
            else:
                self.current_action = "Neutral" # Reset to Neutral when in the dead zone

            # --- Jump/Duck (Shoulder Movement) ---
            shoulder_y = (landmarks[self.mp_pose.PoseLandmark.LEFT_SHOULDER].y +
                           landmarks[self.mp_pose.PoseLandmark.RIGHT_SHOULDER].y) / 2
            self.smoothed_shoulder_y = (self.smoothing_factor * shoulder_y) + ((1 - self.smoothing_factor) * self.smoothed_shoulder_y)
            current_time = time.time()
            
            if (self.smoothed_shoulder_y < self.neutral_shoulder_y - self.jump_threshold/self.frame_height) and (current_time - self.last_jump_time > self.jump_debounce_time) :
                self.current_action = "Jump"
                pyautogui.press('up')
                self.last_jump_time = current_time
            elif (self.smoothed_shoulder_y > self.neutral_shoulder_y + self.duck_threshold/self.frame_height) and (current_time-self.last_duck_time > self.duck_debounce_time):
                self.current_action = "Duck"
                pyautogui.press("down")
                self.last_duck_time = current_time
        else:
            self.current_action = "Neutral"  # No reliable pose, so no action
            

    def run(self):
        try:
            self.calibrate()
            # Attempt to bring the game window to the foreground (best-effort)
            pyautogui.getWindowsWithTitle("Subway Surfers")[0].activate()
            time.sleep(0.5) # Allow time for window switch.

            while True:
                ret, frame = self.cap.read()
                if not ret:
                    print("Error: Cannot read frame.")
                    break

                frame = cv2.flip(frame, 1)  # Mirror the frame
                processed_frame = self.process_frame(frame)

                cv2.imshow('Pose Controlled Subway Surfers', processed_frame)

                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break
        except Exception as e:
            print(f"An error occurred: {e}")
        finally:
            self.cap.release()
            cv2.destroyAllWindows()

if __name__ == "__main__":
    controller = PoseController()
    controller.run()

pip install opencv-python mediapipe pyautogui numpy

